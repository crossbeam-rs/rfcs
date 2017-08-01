# Summary

This is a proposal for clarifying the synchronization performed in the epoch-based memory
reclamation scheme, and explaining why Crossbeam's implementation is correct
w.r.t. C11/C++11/LLVM/Rust relaxed-memory concurrency model (the "C11 memory model" from now on).



# Motivation

Crossbeam is based on the epoch-based memory reclamation scheme (EBR).  In EBR, an **epoch** (think:
timestamp) is maintained, and an object is mark with the current epoch when the object is unlinked.
The unlinked object is safe to deallocate later when the epoch is advanced enough.  As you may
guess, the correctness of EBR crucially depends on the invariant on the epoch: unlinked objects are
inaccessible after two advancements.

EBR is introduced in [Keir Fraser's PhD thesis][kfraser], and later Aaron Turon introduced it
in [his blog post on Crossbeam][aturon] to the Rust community.  Both explains EBR in details so that
a reader can understand not only the intuition behind it but also why it is correct.  However, both
assumed we are on the *sequentially consistent (SC) concurrency model*: every access to the memory
from all the threads are *linearizable*, and each load reads from the most-recently stored value
according to the linearization order.  However, unfortunately, the real-world doesn't allow us to
enjoy the luxury of SC.  In particular, the C11/C++11/LLVM/Rust memory model is *relaxed*, or
*weak*, allowing each load to read from stale old values.

As a result, it is still unclear after two years of development whether the implementation of
Crossbeam is correct w.r.t. the C11 memory model.  We believe that in order for Crossbeam to indeed
serve as the crossbeam for Rust concurrency, as `java.util.concurrent` does for Java concurrency,
Crossbeam's correctness should be formally analyzed and hopefully verified.  This RFC aims for
providing a step towards the goal.

## Non-motivation

Crossbeam is the marriage of EBR and Rust's linear type system with lifetime.  However, this RFC
does not aim for clarifying how Rust's type system helps to ensure the correctness of Crossbeam.



# Detailed design

In this section, we present the proposed implementation of Crossbeam in pseudocode, and explain why
it is correct w.r.t. the C11 memory model.


## Proposed implementation

As discussed in Aaron Turon's original [blog post][aturon]
and [the pull request of a previous RFC][rfc2-pr], Crossbeam guarantees that *every access to an
object should happen before the object is deallocated*.  In order to provide this guarantee,
Crossbeam assumes that when a pinned thread marks an object as unlinked, *the thread already removed
all the references to the object from the memory, and all the other threads do not reintroduce a
reference to the object in the memory*; and Crossbeam defers the deallocation of an unlinked object
until all the threads concurrently pinned at the time the object is unlinked, are unpinned.

In order to track the concurrently pinned threads, Crossbeam utilizes the epoch as follows:

- There is a local epoch `th.epoch` for each *pinned* thread `th`, and a global epoch `EPOCH`.

- When an object is deallocated, it is marked with the **current** global epoch, as in the following
  pseudocode (fences and orderings such as `Relaxed`, `Acquire`, `Release`, and `SeqCst` will be
  explained below):

  ```rust
  fn unlink(l) {
    'L10: atomic::fence(SeqCst);
    'L11: let epoch = EPOCH.load(Relaxed);

    'L12: // mark `l` with `epoch`
  }
  ```

- Local epoch `th.epoch` is assigned the value of the global epoch `EPOCH` at the time the
  corresponding thread `th` is pinned, as follows:

  ```rust
  fn pin(th) {
    'L20: let e = EPOCH.load(Relaxed);
    'L21: th.epoch.store(e, Relaxed);
    'L22: atomic::fence(SeqCst);

    'L23: // now objects marked with `<= e - 2` can be deallocated.
  }
  ```
  
  When a thread is unpinned, its local epoch is set to a sentinel value, as follows:

  ```rust
  fn unpin(th) {
    'L30: th.epoch.store(SENTINEL, Release);
  }
  ```

- The global epoch `EPOCH` is incremented only when all the local epochs of pinned threads equal to
  the global epoch, as follows:

  ```rust
  fn try_advance() {
    'L40: let G = EPOCH.load(Relaxed);
    'L41: atomic::fence(SeqCst);

    'L42: for th in threads {
    'L43:   let e = th.epoch.load(Relaxed);
    'L44:   if e != SENTINEL && e != G { return; }
    'L45: }
    'L46: atomic::fence(Acquire);

    'L47: EPOCH.store(G + 1, Release);

    'L48: // now objects marked with `<= e - 1` can be deallocated.
  }
  ```

Note that at `pin(): 'L23` and `try_advance(): 'L48`, an unlinked object marked with an epoch two
generations before than the current global one, can be deallocated.  In other words, you can
deallocate an object marked with `X` when the current global epoch `>= X+2`.


## Correctness

Now we explain why the proposed implementation is correct w.r.t. the C11 memory model.

Suppose that the current epoch is `X+2`, and thread U `unlink()`ed an object and marked it with the
epoch `X`, and thread D is about to deallocate the object.  For correctness, we need to ensure that
the accesses to the object by `pin()`ned threads happen before the deallocation.  For example, in
the following timeline, threads A and B should not be able to access `l` after its deallocation.

```
         [X]                   [X+1]              [X+2]
EPOCH    +---------------------+----------------+-------------------------------

                                                      [deallocating l]
Thread D ---------------------------------------------+-------------------------

                                          [2. pinned]         [is l accessible?]
Thread A ---------------------------------+-------------------+-----------------

      [l removed] [1. unlinking l]   [unpinned]
Thread U ---------+------------------+------------------------------------------

             [3. pinned]  [unpinned]                          [is l accessible?]
Thread B ----+------------+-----------------------------------+-----------------

                                                [4. try_advance]
Thread E ---------------------------------------+-------------------------------
```

As we will see, the correctness heavily depends on the "cumulativity" of SC fences.  Intuitively, it
means that if a thread A's SC fence is performed before another thread B's SC fence, then all
information gathered before A's fence becomes visible after B's fence.  In that case, we say that
the instructions before A's fence is *visible via SC fences to* the instructions after B's fence.
This synchronization is also used in the [C11 version of Chase-Lev deque][weak-chase-lev].  For more
information on the cumulativity, see a [recent understanding of C11 SC atomics][scfix].

Now we consider two cases on the order of `unlink()`'s and `pin()`'s SC fence.

### When `unlink()`'s SC fence is performed before `pin()`'s SC fence

Thread A, for example, should not be able to access `l` after its deallocation, because:

- By assumption, the `unlink()`ing thread already removed all the references to the object from the
memory on its point of view.  By cumulativity form `unlink()`'s SC fence (1. in the timeline) to
`pin()`'s (2. in the timeline), `pin()`ned thread cannot access the old object `l`.


### When `pin()`'s SC fence is performed before `unlink()`'s SC fence

Thread B, for example, should not be able to access `l` after its deallocation, because:

- Let's consider the invocation of `try_advance()` that actually advances the global epoch to `X+2`.
  `unlink()`'s SC fence (1. in the timeline) is performed before `try_advance()`'s SC fence (4. in
  the timeline).  Because otherwise, `try_advance(): 'L40`, which reads `X+1` from `EPOCH`, is
  visible via SC fences to `unlink(): 'L11`, which reads `X` from `EPOCH`.  This is a contradiction.

- By transitivity, `pin()`'s SC fence (3. in the timeline) is performed before `try_advance()`'s SC
  fence (4. in the timeline).  Thus the store to the local epoch at `pin(): 'L21` is visible via SC
  fences to the load from it at `try_advance(): 'L43`, and `'L43` should read from what is written
  at `'L21` or a more recent value than that.

- In order to reach `try_advance(): 'L47` and increment the global epoch, `'L43` should not read
  from what is written at `'L21`.  Thus it should read from what is written at `unpin(): 'L30` or a
  more recent value than that, and by release-acquire synchronization, `'L30` happens before `'L46`.
  
- Since a thread should access an object before `unpin()`, every access to the unlinked object from
  the `pin()`ned thread happens before the object's deallocation.



# Alternatives

## The current implementation

A reader may notice that the pseudocode differs from the current implementation of Crossbeam.  We
believe the current implementation is buggy w.r.t. the C11 memory model, because it uses SC
reads/writes that do not provide cumulativity.  As opposed to widely believed, SC loads and stores
are relatively weak according to the C11 standard: in particular, the memory model designers intend
to allow reordering of an SC load before a relaxed store, and that of SC writes after a relaxed
load.  See a [recent paper][scfix] for more details.

## Target-dependent implementation

Alternatively we can write the core of Crossbeam for each target architecture.  It may be beneficial
for the performance: it is unclear whether the proposed implementation is compiled to the most
efficient implementation for each target architecture.  A downside is targeting each architecture
will cost a lot.

## Waiting for the memory models to be stabilized

Before start trying to formally explain why Crossbeam is correct, it may be worth waiting for the
underlying C11 memory model to be stabilized.  The model has greatly been clarified and improved in
the last decades, but it is still subject to change a lot.  However, the fragment of the model on
which Crossbeam relies is quite solid so that it is worth reasoning Crossbeam on the
current [state-of-art concurrency semantics][promising].

## Ensuring correctness by testing

In order to ensure the correctness of Crossbeam's implementation, one may thoroughly test it using
real-world workloads.  However, as [Dijkstra pointed out][dijkstra] in the early days, "testing
shows the presence, not the absence of bugs".



# Unresolved questions

Is it *really* the most efficient implementation in the C11 memory model?

Is there any soundness gap in above description?  More ambitiously, can we formally verify the
correctness of Crossbeam?


[dijkstra]: http://homepages.cs.ncl.ac.uk/brian.randell/NATO/nato1969.PDF
[promising]: http://sf.snu.ac.kr/promise-concurrency/
[scfix]: http://plv.mpi-sws.org/scfix/
[aturon]: https://aturon.github.io/blog/2015/08/27/epoch/
[rfc2-pr]: https://github.com/crossbeam-rs/rfcs/pull/2
[kfraser]: https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-579.pdf
[weak-chase-lev]: http://www.di.ens.fr/~zappa/readings/ppopp13.pdf

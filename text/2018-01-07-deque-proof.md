# Summary

This RFC proposes an implementation of work-stealing double-ended queue (deque) on Crossbeam, and
its correctness proof. this RFC has a [companion PR to `crossbeam-deque`][deque-pr].



# Motivation

We believe that the current implementation of deque is not fully optimized yet, leaving
non-negligible room for performance improvement. However, it is unclear how to further optimize the
current implementation while guaranteeing its correctness. We aim to resolve this issue by proving
an optimized implementation.



# Detailed design

In this section, we present a pseudocode of deque, and prove that (1) it is safe in the C/C++ memory
consistency model, and (2) it is functionally correct, i.e. it acts like a deque.


## Semantics of C/C++ relaxed-memory concurrency

For the semantics of C/C++ relaxed-memory concurrency on which the deque implementation is based, we
use the state-of-art [promising semantics][promising]. For a gentle introduction to the semantics,
we refer the reader to [this blog post][synch-patterns].

One caveat of the original promising semantics is its lack of support for memory allocation. In this
RFC, we use the following semantics for memory allocation:

- In the original promising semantics, a timestamp was a non-negative quotient number: `Timestamp =
  Q+`. Now we define `Timestamp = Option<Q+>`, where `None` means the location is
  unallocated. Suppose `None < Some(t)` for all `t`.

- At the beginning, the thread's view is `None` for all the locations. When a thread allocates a
  location, its view on the location becomes `0`.

- If a thread is reading a location and it's view on the location is `None`, i.e. it is unallocated
  in the thread's point of view, the behavior is undefined.

We believe this definition is compatible with all the results presented in the [paper][promising] on
the promising semantics. We hope [the Coq formalization][promising-coq] of promising semantics be
updated accordingly soon.


## Proposed implementation

A work-stealing deque is owned by the "owner" thread, which can push items to and pop items from the
"bottom" end of the deque. Other threads, which we call "stealers", try to steal items from the
"top" end of the deque. The following is a summary of its interface:

```rust
impl<T: Send> Deque<T> { // Send + !Sync
    pub fn new() -> Self { ... }
    pub fn stealer(&self) -> Stealer<T> { ... }

    pub fn push(&self, value: T) { ... }
    pub fn pop(&self) -> Option<T> { ... } // None if empty
}

impl<T: Send> Stealer<T> { // Send + Sync
    pub fn steal(&self) -> Option<T> { ... }
}
```

Chase and Lev proposed [an efficient implementation of deque][chase-lev] on sequential consistency,
and Lê et. al. proposed [its variant for ARM and C/C++ relaxed-memory
concurrency][chase-lev-weak]. This RFC presents an even more optimized variant for C/C++.

Chase and Lev's implementation of deque consists of (1) the `bottom` index for the worker, (2) the
`top` index for both the worker and the stealers, and (3) the underlying `buffer` that contains
items. When the buffer is too small or too large, the worker may replace it with one with more
appropriate size:

```rust
struct<T: Send> Inner<T> {
    bottom: AtomicUsize,
    top: AtomicUsize,
    buffer: Ptr<Buffer<T>>,
}

impl<T: Send> Inner<T> {
    ...
}
```

The `Deque` and `Stealer` structs have a counted reference to the implementation, and delegate
method invocations to it:

```rust
struct<T: Send> Deque<T> {
    inner: Arc<Inner<T>>,
}

impl<T:Send> Deque<T> {
    fn push(...) {
        self.inner.push(...)
    }

    ...
}

...
```

The following is pseudocode for `Inner<T>`'s `fn push()`, `fn pop()`, and `fn steal()`:

```rust
pub fn push(&self, t: T) {
    'L101: let b = self.bottom.load(Relaxed);
    'L102: let t = self.top.load(Acquire);
    'L103: let mut buffer = self.buffer.load(Relaxed, epoch::unprotected());

    'L104: let cap = buffer.get_capacity();
    'L105: if b - t >= cap {
    'L106:     self.resize(cap * 2);
    'L107:     buffer = self.buffer.load(Relaxed, epoch::unprotected());
    'L108: }

    'L109: buffer.write(b % buffer.get_capacity(), t);
    'L110: self.bottom.store(b + 1, Release);
}

pub fn pop(&self) -> Option<T> {
    'L201: let b = self.bottom.load(Relaxed);
    'L202: self.bottom.store(b - 1, Relaxed);
    'L203: fence(SeqCst);
    'L204: let t = self.top.load(Relaxed);
    'L205: let len = b - t;

    'L206: if len <= 0 {
    'L207:     self.bottom.store(b, Relaxed);
    'L208:     return None;
    'L209: }

    'L210: let buffer = self.buffer.load(Relaxed, epoch::unprotected());
    'L211: let mut value = Some(buffer.read((b - 1) % buffer.get_capacity()));

    'L212: if len == 1 {
    'L213:     if self.top.compare_and_swap(t, t + 1, Release, Acquire).is_err() {
    'L214:         mem::forget(value.take());
    'L215:     }
    'L216:     self.bottom.store(b, Relaxed);
    'L217: } else {
    'L218:     let cap = buffer.get_capacity();
    'L219:     if len < cap / 4 {
    'L220:         self.resize(cap / 2);
    'L221:     }
    'L221: }

    'L222: value
}

fn resize(&self, cap_new) {
    'L301: let b = self.bottom.load(Relaxed);
    'L302: let t = self.top.load(Relaxed);
    'L303: let old = self.buffer.load(Relaxed, epoch::unprotected());
    'L304: let new = Buffer::new(cap_new);

    'L305: ... // copy data from old to new

    'L306: let guard = epoch::pin();
    'L307: self.buffer.store(Owned::new(new).into_shared(guard), Release);
    'L308: guard.defer(move || old.into_owned());
}

pub fn steal(&self) -> Option<T> {
    'L401: let mut t = self.top.load(Relaxed);

    'L402: let guard = epoch::pin_fence(); // epoch::pin(), but forces fence(SeqCst)
    'L403: loop {
    'L404:     let b = self.bottom.load(Acquire);
    'L405:     if b - t <= 0 {
    'L406:         return None;
    'L407:     }

    'L408:     let buffer = self.buffer.load(Acquire, &guard);
    'L409:     let value = buffer.read(t % buffer.get_capacity());

    'L410:     match self.top.compare_and_swap_weak(t, t + 1, Release) {
    'L411:         Ok(_) => return Some(value),
    'L412:         Err(t_old) => {
    'L413:             mem::forget(value);
    'L414:             t = t_old;
    'L415:             fence(SeqCst);
    'L416:         }
    'L417:     }
    'L418: }
}
```


## Safety

We first prove that the implementation is memory-safe, i.e. it does not invoke undefined behavior
due to illegal memory accesses. We discuss two possible causes of undefined behavior: data race and
invalid pointer dereference.


### Presence of Data Races

In the C/C++ standard, if two threads race on a non-atomic object, i.e. they concurrently access it
and at least one of them writes to it, then the program's behavior is undefined. Unfortunately, in
fact, `fn push(): 'L107` and `fn steal(): 'L408` may race on `buffer`. For example, the scheduler
may stop a `steal()` invocation right after `'L402` so that `t` read in `'L401` may be arbitrarily
stale. Now, suppose that in a concurrent `push()` invocation, `b` equals to `t +
buffer.get_capacity()` and it is overwriting a value to `buffer`'s `(b % buffer.get_capacity())`-th
element. At the same time, the `steal()` invocation may wake up and read `buffer`'s `(t %
buffer.get_capacity())`-th element, incurring a data race with the `push()`.

In order to avoid this data race, we may use an atomic buffer. However, the deque's item may be
arbitrarily large, and [atomic accesses to large objects are most likely blocking][cppatomic], which
is highly undesirable. Thus there is a genuine data race in Chase and Lev's deque w.r.t. the C/C++
standards.

We make a rather bold claim: **data races are NOT undefined behavior**, but a racing reader may read
an unspecified value (or the `poison` value in the LLVM lingo), and racing writers may write each
word of values in any order. This is the specification on racy accesses of promising semantics,
except that the semantics currently doesn't specify the behavior of multi-word accesses.


### Absence of Invalid Pointer Dereference

<!-- By invalid pointer dereference we mean (1) null pointer dereference, (2) use-after-free, or (3)
free-after-free. -->

It boils down to proving the safety of every access to the buffer pointed-to by `Inner`'s
`self.buffer`, (1) since the other shared objects, namely `self.bottom` and `self.top`, are just
integers, and (2) accesses to thread-local storage are straightforwardly safe. It is worth noting
that while the deque is shared among the owner (`Deque<T>`) and its stealers, only the owner is able
to call `fn push()`, `fn pop()`, and `fn resize()`, thereby modifying `self.bottom` and
`self.buffer`.

After `self.buffer` is initialized, it is modified only by the owner in `fn resize(): 'L307`. Since
elements inside a buffer is always accessed modulo the buffer's capacity (`'L109`, `'L211`, `'L409`)
and the buffer's size is always nonzero, there is no buffer overrun.

Thus it remains to prove that the buffer is not used after freed. Thanks to Crossbeam, we don't need
to take care of all the details of memory reclamation; as a user of Crossbeam, we only need to prove
that:

- When a buffer is deferred to be dropped (`'L308`), there is no longer a reference to the buffer in
  the shared memory, and no concurrent mutator introduces the reference to the memory again.

  You can easily check these properties because the owner basically "owns" the buffer: only the
  owner introduces/eliminates references to a buffer to/from the shared memory. In particular,
  before a buffer is deferred to be dropped at `'L308`, the only reference to the buffer via
  `self.buffer` is removed at `'L307`.

- With an `unprotected()` guard, we should not defer to drop an object that another thread is
  accessing, and should not access an object that another thread is deferring to drop.

  It is indeed the case because (1) a buffer is deferred to be dropped only by the owner with a
  protected guard, and (2) stealers do not use an unprotected guard.



## Functional Correctness

A more interesting question is: does it act like a deque?

### Specification: Linearizability and Synchronization

In order to answer the question, we first define the sequential specification of a deque. A *deque
state* as a triple `(t, b, A)`, where `t` is the top index, `b` is the bottom index, and `A` is the
array of the values within the indexes `[t, b)`. A `push()` method invocation increases the bottom
index `b`, and push the input value at the end of `A`. A `pop()` method invocation decreases the
bottom index `b` and pop the last value from `A`, if `t < b`; otherwise, touch nothing and return
`EMPTY`. (As a special case, if `t+1 = b`, you can increase the top index `t` instead of decrease
the bottom index `b`.) A `steal()` method invocation increases the top index `t` and pop the first
value from `A`, if `t < b`; otherwise, touch nothing and return `EMPTY`. By `S -(I,R)-> S'` we
denote the described *transition relation* from a deque state `S` into `S'` by the method invocation
`I` returning `R`.

A shared object is **linearizable** as a deque if the following holds:

> Suppose that multiple threads invoked methods (i.e. `push()`, `pop()`, `steal()`) on the shared
> object in a single execution. For those invocations that is not `steal()` returning `RETRY`, there
> exists a permutation `I_1`, ..., `I_n` of them, which we call **"linearization order"**, denoted
> by `<`, and `(t_i, b_i, A_i)` for each `i` such that:
>
> - (VIEW) For all `x` and `y`, if `I_x -view-> I_y`, then `x < y` holds;
>
> - (SEQ) For all `i`, `(t_i, b_i, A_i) -(signature(I_i),ret(I_i))-> (t_(i+1), b_(i+1), A_(i+1))`
>   holds. In other words, the invocations `I_1`, ..., `I_n` return outputs **as if** the methods
>   are sequentially (i.e. non-concurrently) executed to a deque one-by-one in the linearization
>   order.

Here, by `view_beginning(j)` and `view_end(j)` we mean the thread `j`'s view at the beginning and
the end of the invocation. A view `v1` is less than or equal to another view `v2`, denoted by `v1 <=
v2`, if for any location `l`, `v1[l] <= v2[l]` holds; and `v1` is less than `v2`, denoted by `v1 <
v2`, if (1) `v1 <= v2`, and (2) there exists a location `l` such that `v1[l] < v2[l]`. By `j_x
-view-> j_y` we mean `view_end(j_x) < view_beginning(j_y)`. Also, `signature(I_i)` and `ret(I_i)`
are the function call signature and the return value of `I_i`.

In addition to linearizability, we require that a matching pair of `push()` and `steal()`
synchronize like a matching pair of a release-store and an acquire-load:

> - (SYNC) If a value is `push()`ed by an invocation `I` and then `steal()`ed by `J`, then
> `view_beginning(I) <= view_end(J)` holds.

Note that a similar property for a matching pair of `push()` and `pop()` is a corollary of
linearizability, as only the owner is able to call `push()` and `pop()` and the `push()` should be
called before the `pop()` in the program order.


#### Comparison of the Definition of Linearizability

The condition `(VIEW)` and `(SEQ)` is a view-oriented interpretation of a similar condition in the
[the traditional definition of linearizability][linearizability]. What is unique in `(VIEW)` is that
it cannot properly distinguish invocations with the same view. For example, suppose the client code
looks like:

```rust
fn client() {
    i_1();
    i_2();
}
```

and `view_beginning(i_1) = view_end(i_1) = view_beginning(i_2) = view_end(i_2)`. By definition, `i_2
-> i_1` is a legit linearization order, yet morally `i_1` should be ordered before `i_2` because it
is so in the _program order_ (think: the execution order in a single thread). However, we do not
regard it a problem: if `i_1` and `i_2` do not change the view, they are read-only, and should not
change the state of the shared object. Thus it doesn't matter whether `i_1` is considered to be
before or after `i_2`.

<!-- If we want to be pedantic, we may alternatively require every method to write something to a -->
<!-- thread-local storage, effectively disallowing that the views at the beginning and the end should -->
<!-- differ. We can use this storage only in the proof, and optimize out in the runtime. -->


### Construction of Linearization Order

Consider the invocations to the deque in an execution. For the owner's invocations, the program
order is the only possible linearization among them. Let's assume that `O_0`, ..., `O_(o-1)` is the
owner's invocations, sorted according to the program order. We construct a full linearization order
by inserting stealers' invocations into it.

Note that every owner's invocation writes to `bottom`, and every stealers' invocation reads from
`bottom`. We classify the stealers' invocations into "groups" `{G_i}` as follows:

- For a stealer's invocation `S`, if `S` reads from the initial value of `bottom`, then `S ∈
  G_(-1)`.

- Otherwise, let `O_i` be the owner's invocation that writes to `bottom` a value which `S` read.

- If `O_i` is not a `pop()` taking the "irregular" path (i.e. not entering `'L218`) and `S` returns
  a value, then `S ∈ G_(i-1)`; otherwise, then `S ∈ G_i`.

We will insert the invocations in `G_i` between `O_i` and `O_(i+1)`. Inside a group `G_i`, we give
the linearization order as follows:

- Let `STEAL^x` be the set of steal invocations in which an element at the index `x` is stolen, and
  `STEAL_EMPTY` be the set of steal invocations in which a stealer fails to steal an element because
  the deque is empty. Let `STEAL` be `union_x {STEAL^x}`.

- Within `G_i`, place `STEAL` invocations first, and then `STEAL_EMPTY` invocations.

- If there are multiple `STEAL` invocations, order `STEAL^x` before `STEAL^y` if `x < y`.

- If there are multiple `STEAL_EMPTY` invocations, order `I` before `J` if `I -view-> J`. (Note that
  `-view->` relation is acyclic; and there can be multiple possible linearizations.)


### Auxiliary Lemma

- > Let `WF_i` and `WL_i` be `O_i`'s first and last write to `bottom`.

- > For arbitrary `i`, since `O_(i-1)` wrote `WF_(i-1), WL_(i-1)` and `O_i` wrote `WF_i, WL_i`, we
  have `Timestamp(WL_(i-1)) <= view_beginning(O_i)[bottom] < Timestamp(WF_i)`. Similarly, we have
  `Timestamp(WL_i) <= view_end(O_i)[bottom] < Timestamp(WF_(i+1))`.

- > For arbitrary `i` and `S ∈ G_i`, since `S` read `WF_i` or a later value, we have
  `Timestamp(WF_i) <= view_end(S)[bottom]`. Also, since `S` read `WL_(i+1)` or an earlier value, we
  have `view_beginning(S)[bottom] <= Timestamp(WL_(i+1))`.

- > Suppose `O_i` is `pop()` taking the irregular path. Let `x` be the value of `bottom` read by
  `O_i` at `'L201`. Then [timestamp of `top = x`] <= `view_end(O_(i+1))[top]`.

  Let `y` be the value of `top` read by `O_i` at `'L204`. Since `O_i` is taking the irregular path,
  we have `x-1 <= y`. If `x <= y`, then the conclusion trivially follows. If `y = x-1`, then `O_i`
  tries to compare-and-swap (CAS) `top` from `y` to `y+1` at `'L213`. Regardless of whether the CAS
  succeeds, `O_i` reads or writes a value of `y >= x`.

  It is worth nothing that for this lemma to hold, it is necessary for the CAS at `'L213` to be
  strong.

- > (STEAL-IRREGULAR): Suppose `O_i` is `pop()` taking the irregular path, and `S` is `steal()` that
  reads from `bottom` a value written by `O_i` and returns a value. Let `x` be the value of `bottom`
  read by `O_i` at `'L201`, and `y` be the value of `top` written by `S` at `'L410`. Then `y < x`
  holds.

  If `S` read `x-1` from `bottom`, then `y <= x-1`. Now suppose `S` read from `bottom = x` written
  by `O_i` at `'L207` (or `'L216`). We already know `y <= x`, so suppose `y = x` and let's derive a
  contradiction.

  + Case `O_i` read `top = x` at `'L204`.

    In order for `S` to read `bottom = x` at `'L404` and `O_i` to read `top >= x` at `'L204` at the
    same time, either `S`'s write to `top` at `'L410` should be promised before reading `bottom` at
    `'L404` or `O_i`'s write to `bottom` at `'L207` should be promised before reading `top` at
    `'L204`. But this is impossible: `S`'s write to `top` at `'L410` is a release-store, and `O_i`s
    write to `bottom` at `'L207` has a control dependency to the read from `top` at `'L204`.

  + Case `O_i` read `top >= x` at `'L213`.

    Then there is a release-acquire synchronization from `S`'s write to `top` at `'L410` to `O_i`'s
    read from `top` at `'L213`, so it is impossible for `S` to read `bottom = x` written by `O_i` at
    `'L216`.


### Proof of `(VIEW)`

For `(VIEW)`, it is sufficient to prove that:

1. `(VIEW-OWNER-STEALER)`: for all `i <= j` and `S ∈ G_j`, it is not the case that `S -view-> O_i`.
2. `(VIEW-STEALER-OWNER)`: for all `i < j` and `S ∈ G_i`, it is not the case that `O_j -view-> S`.
3. `(VIEW-STEALER-INTER-GROUP)`: for all `i < j`, `S_i ∈ G_i` and `S_j ∈ G_j`, it is not the case
   that `S_j -view-> S_i`.
4. `(VIEW-STEALER-INTRA-GROUP)`: for all `i` and `S, S' ∈ G_i`, if `S` is placed before `S'`, then
   it is not the case that `S' -view-> S`.

(Note that what would be `(VIEW-OWNER-OWNER)` is obvious.)

#### Proof of `(VIEW-OWNER-STEALER)`.

Since `view_beginning(O_i)[bottom] < Timestamp(WF_i) <= Timestamp(WF_j) <= view_end(S)[bottom]`, it
is not the case that `S -view-> O_i`.

#### Proof of `(VIEW-STEALER-OWNER)`.

We have `view_beginning(S)[bottom] <= Timestamp(WL_(i+1)) <= Timestamp(WL_j) <=
view_end(O_j)[bottom]`. If `view_beginning(S)[bottom] < view_end(O_j)[bottom]`, then it is not the
case that `O_j -view-> S`.

Now suppose otherwise. Then we have `j = i+1` and `view_beginning(S)[bottom] = Timestamp(WL_(i+1)) =
view_end(O_(i+1))[bottom]`, so `S` read `bottom` from `WL_(i+1)` and returns a value and `O_(i+1)`
is `pop()` taking the irregular path. Let `x` be the value `O_(i+1)` read from `bottom` at
`'L201`. Since `S` wrote a value `<= x` to `top` and `O_(i+1)` read a value `>= x` from `top`, have
`view_beginning(S)[top]` < [the timestamp of `top = x`] <= `view_end(O_(i+1))[top]`. So it is not
the case that `O_(i+1) -view-> S`.

#### Proof of `(VIEW-STEALER-INTER-GROUP)`.

We have `view_beginning(S_i)[bottom] <= Timestamp(WL_(i+1))` and `Timestamp(WF_j) <=
view_end(S_j)[bottom]`. Thus if either `i+1 < j`, `view_beginning(S_i)[bottom] <
Timestamp(WF_(i+1))` or `Timestamp(WL_j) < view_end(S_j)[bottom]` holds, then it is not the case
that `S_j -view-> S_i`.

Now suppose otherwise. Then `j = i+1`, `O_(i+1)` is `pop()` taking the irregular path, `S_i` and
`S_j` both read from `bottom` values written in `O_(i+1)`, `S_i` returns a value, and `S_j` returns
`EMPTY`. Let `x` be the value of `bottom` read by `O_(i+1)` at `'L201`, and `y` be the value of
`top` written by `S`. Then `y < x` holds by an auxiliary lemma. So we have `view_beginning(S_i)[top]
< Timestamp(top, y) <= Timestamp(top, x-1) <= view_end(S_j)[top]`, and it is not the case that `S_j
-view-> S_i`.

#### Proof of `(VIEW-STEALER-INTRA-GROUP)`.

- Case 1: `S, S' ∈ STEAL`.

  Suppose `S` read the value `y` from `top` at `'L401`, and `S'` read the value `w` from `top` at
  `'L401`. By the construction, `y < w`. Thus it is not the case that `S' -view-> S`.

- Case 2: `S, S' ∈ STEAL_EMPTY`. Obvious from the construction.

- Case 3: `S ∈ STEAL` and `S' ∈ STEAL_EMPTY`.

  Let `b` be the value of `WL_i`; `x` (and `x'`) be the values `S` (and `S'`, respectively) read
  from `bottom` at `'L404`; and `y` (and `y'`) be the values `S` (and `S'`, respectively) read from
  `top` at `'L401`. Since `S' ∈ STEAL`, we have `y < x`. Since `S' ∈ STEAL_EMPTY`, we have `b-1 <=
  x' <= b` and `x' <= y'`.

  It is sufficient to prove that `y < y'` holds. From `S ∈ STEAL`, we know that either (1) `S` read
  a value from `bottom` written by `O_(i+1)`, and `O_(i+1)` is `pop()` taking the irregular path; or
  (2) `S` read a value from `bottom` written by `O_i`, and `O_i` is not `pop()` taking the irregular
  path. If the former is the case, then `y+1 < b` so that `y <= b-2 < b-1 <= x' <= y'`. If the
  latter is the case, then `y < x = b = x' <= y'`.


## Proof of `(SEQ)` and `(SYNC)`

Let `I_0`, ..., `I_(n-1)` be the invocations sorted according to the constructed linearization
order. In addition to `(SEQ)` and `(SYNC)`, we will simultaneously prove the following conditions
with the same existentially quantified values of `t_i, b_i, A_i`:

> `(CONTENT)`: for all `i` and `x ∈ [t_i, b_i)`, there exists a `push()` invocation into `x` in
> `I_0, ..., I_(i-1)`; and `A_i[x]` is the value inserted by the last such invocation.
>
> `(TOP)`: `t_i` equals to the value of the last write to `top` in `I_0`, ..., `I_(i-1)`.

We prove that `{I_i}` satisfies `(SEQ)`, `(SYNC)`, `(CONTENT)`, and `(TOP)` by induction on `n`:
suppose `I_0`, ..., `I_(i-1)` satisfies those conditions, and let's prove that `I_i` also satisfies
those conditions. We prove for each case of `I_i`.

- Case 1: `I_i` is `push()`.

  Quite obvious.

- Case 2: `I_i` is `pop()` no taking the irregular path.

  Let `x` and `y` be the values `I_i` read from `bottom` at `'L201` and `top` at `'L204`,
  respectively. Then we have `x = b_i`, as `bottom` is modified only by the writer. Since `I_i` is
  not taking the irregular path, we have `y+2 <= x`. We also prove `t_i <= y+1`. Consider the
  invocation `I` that writes `y+2` to `top`, if exists. (Otherwise, `t_i <= y+1` should obviously
  hold.) If `I` is `pop()`, then `I` should be linearized after `I_i` thanks to the coherence of
  `top`; if `I` is `steal()`, by the synchronization of the SC fences from `I_i` to `I` via `top`,
  `I` should load a value that is coherence-after-or `WL_j` from `bottom`, where `j` is such an
  index that `I_i = O_j`. Thus `I` is linearized after `I_i`, and `t_i <= y+1` holds.

  Then we have `t_i <= y+1 < y+2 <= x = b_i`, and it is legit to pop a value from the bottom end of
  the deque and decrease `bottom`.

  FIXME(todo): the right value?

- Case 3: `I_i` is `pop()` taking the irregular path.

  Let `x` and `y` be the values `I_i` read from `bottom` at `'L201` and `top` at `'L204`,
  respectively. Similarly to the above case, we have `x = b_i`. Since `I_i` takes the irregular
  path, we have `y >= x-1`.

  + Case `y >= x`.

    We prove `x <= t_i` as follows. Consider the invocation `I` that writes `x` to `top`. If `I` is
    `pop()`, then `I` should be linearized before `I_i` thanks to the coherence of `top`. Suppose
    `I` is `steal()`. Let `W` be the message `I` read from `bottom` at `'L404`. Then since `I`
    returns a value, we have `x <= Value(W)`. It is sufficient to prove that `W <= WL_j` in the
    coherence order, where `j` is such an index that `I_i = O_j`. Let's suppose otherwise. If
    `O_(j+1)` is `pop()`, then `W ≠ WF_(j+1)` since `Value(WF_(j+1)) = x-1`. Otherwise, `W` is
    either a release-store, or there is an SC fence between `I_i`'s read from `top` at `'L204` and
    `W`. But this is impossible: in order for `I` to read from `W` at `'L404` and `I_i` to read `top
    >= x` at `'L204` at the same time, either `I`'s write to `top` at `'L410` should be promised
    before reading `bottom` at `'L404` or `W` should be promised before `I_i` reading `top` at
    `'L204`, but the former is a release-store and the latter is either a release-store or after an
    SC fence.

    Thus we have `b_i = x <= t_i`, and it is legit for `I_i` to go to `'L207`, restore the original
    value of `bottom`, and return `EMPTY`.

    <!-- [I] -->
    <!-- R t x-1 -->
    <!-- ------- -->
    <!-- R b >=x -->
    <!-- U t x-1 x -->

    <!-- [I_i] -->
    <!-- R b x -->
    <!-- W b x-1 -->
    <!-- ------- -->
    <!-- R t >=x -->
    <!-- W b x -->

  + Case `y = x-1`.

    Then `I_i` performs a (strong) compare-and-swap (CAS) at `'L213`.

    If the CAS fails, then `I_i` reads `top >= x` at `'L213` and writes `bottom = x` at
    `'L216`. Let's prove that `x <= t_i`. Consider the invocation `I` that writes `x` to `top`. If
    `I` is `pop()`, then `I` should be linearized before `I_i` thanks to the coherence of
    `top`. Suppose `I` is `steal()`. Then there is a release-acquire synchronization from `I`'s
    write to `top` at `'L410` to `I_i`'s read from `top` at `'L213`, and `I` should load a value
    that is coherence-before `WL_j` from `bottom`, where `j` is such an index that `I_i = O_j`. Thus
    `I` is linearized before `I_i`, and `x <= t_i` holds. Thus we have `b_i = x <= t_i`, and it is
    legit for `I_i` to return `EMPTY`.
    
    If the CAS succeeds, then `I_i` updates `top` from `x-1` to `x` at `'L213` and writes `bottom =
    x` at `'L216`. Let's prove that `x-1 <= t_i`. Consider the invocation `I` that writes `x-1` to
    `top`. If `I` is `pop()`, then `I` should be linearized before `I_i` thanks to the coherence of
    `top`. Suppose `I` is `steal()`. Let `W` be the message `I` read from `bottom` at `'L404`. Then
    since `I` returns a value, we have `x-1 <= Value(W)`. It is sufficient to prove that `W <= WL_j`
    in the coherence order, where `j` is such an index that `I_i = O_j`. Let's suppose otherwise. In
    order for `I` to read from `W` at `'L404` and `I_i` to read `top = x-1` at `'L204` at the same
    time, either `I`'s write to `top` at `'L410` should be promised before reading `bottom` at
    `'L404` or `W` should be promised before `I_i` reading `top` at `'L204`. But this is impossible:
    the former is a release-store, and the latter (1) has a control dependency to the read from
    `top` at `'L204`, (2) is a release-store, or (3) is after an SC fence.

    Since `I_i` writes `x` to `top`, we have `t_i = x-1`. Thus `t_i = y = x-1 = (b_i)-1`, and it is
    legit to pop the value from the `bottom` end of the deque and increase `top`.

    FIXME(todo): the right value?

    <!-- r t x-2 -->
    <!-- ------- -->
    <!-- r b >=x-1 -->
    <!-- u t x-2 x-1 -->

    <!-- r b x -->
    <!-- w b x-1 -->
    <!-- ------- -->
    <!-- r t x-1 -->
    <!-- u t x-1 x -->
    <!-- w b x -->

    <!-- e.g. -->

    <!-- [steal] -->
    <!-- r t 0 -->
    <!-- ----- -->
    <!-- r b >= 1 -->
    <!-- u t 0 1 -->

    <!-- [pop] -->
    <!-- r b 2 -->
    <!-- w b 1 -->
    <!-- ----- -->
    <!-- r t 1 -->
    <!-- u t 1 2 -->
    <!-- w b 2 -->

    <!-- [pop] -->
    <!-- r b 2 -->
    <!-- w b 1 -->
    <!-- ----- -->
    <!-- r t 2 -->
    <!-- w b 2 -->


- Case 4: `I_i` is `steal()` and it returns a value.

  Let `x` and `y` be the values `I_i` read from `bottom` at `'L404` and `top` at `'L401`,
  respectively. Then either `x = b_i` or `x = (b_i)-1` holds by the definition of the owner methods
  `push()` and `pop()`.

  Let's prove that `y <= t_i`. Consider the invocation `I` that writes `y` to `top`. If `I` is
  `pop()`, then there is a release-acquire synchronization from `I`'s write to `top` at `'L213` to
  `I_i`'s read from `top` at `'L401-'L402`. Let `j` be such an index that `I = O_j`. Then `I_i`
  reads `WF_j` or a later value from `bottom` at `'L404`. From the fact that `I` writes `y` to
  `top`, we know that `Value(WF_j) = y-1` and `Value(WL_j) = y`. Since `I_i` should read `bottom >
  y`, `I_i` reads `WF_(j+1)` or a later value. Thus `I` is linearized before `I_i`, and `y <= t_i`
  holds. If `I` is `steal()`, then there is a release-acquire synchronization from `I`'s write to
  `top` at `'L410` to `I_i`'s read from `top` at `'L401-'L402`. Thus the value `I_i` read from
  `bottom` at `'L404` is the value `I` read from `bottom` at `'L404` or a later value. Thus `I` is
  linearized before `I_i`, and `y <= t_i` holds.

  Since `I_i` writes `y+1` to `top`, we have `t_i = y`. Thus we have `t_i = y <= x-1 = (b_i)-1`, and
  it is legit to steal the value from the `top` end of the deque and increase `top`.

  FIXME(todo): the right value?

  <!-- We also prove `t_i <= y` as follows. Consider the invocation `I` that writes `y+2` to `top`. If -->
  <!-- `I` is `pop()` whose last write to `bottom` is `WL_I`, then `I` should have read `y+1` from `top` -->
  <!-- at `'L213`. By positive piggybacking synchronization from `I_i`'s release-store at `'L410` to -->
  <!-- `I`'s acquire-load at `'L213`, `x` should be a value coherence-before `WL_I`. `I_i` should be -->
  <!-- linearized before `I`, and `t_i <= y` holds. If `I` is `steal()`, by positive piggybacking -->
  <!-- synchronization from `I_i`'s release-store at `'L410` to `I`'s acquire-load at `'L401-'L402`, the -->
  <!-- value `I` read from `bottom` at `'L404` should be coherence-after-or `x`. Since `I_i` writes `y+1` -->
  <!-- to `top` and `I` writes `y+2` to `top`, `I_i` should be linearized before `I`, and `t_i <= y` -->
  <!-- holds. -->


- Case 5: `I_i` is `steal()` and it returns `EMPTY`.

  Let `x` and `y` be the values `I_i` read from `bottom` at `'L404` and `top` at `'L401`,
  respectively. Then `x <= y` holds since `I_i` returns `EMPTY`. Also, either `x = b_i` or `x =
  b_i - 1` holds by the definition of the owner methods `push()` and `pop()`.

  It is sufficient to prove that `b_i <= t_i`, since then it will be legit to return `EMPTY`.

  + Case `x = b_i`.

    We prove `y <= t_i` as follows. Consider the invocation `I` that writes `y` to `top`. If `I` is
    `pop()` and `I = O_j`, then there is a release-acquire synchronization from `I`'s write to `top`
    at `'L213` and `I_i` read from `top` at `'L401-'L402`, and `x` should be coherence-after-or
    `WF_j`. Thus `I` should be linearized before `I_i`, and `y <= t_i`. If `I` is `steal()`, by a
    release-acquire synchronization from `I`'s write to `top` at `'L410` to `I_i`'s read from `top`
    at `'L401-'L402`, `x` should be coherence-after-or the value `I` read from `bottom` at
    `'L404`. Thus `I` should be linearized before `I_i`, and `y <= t_i`.

    Then we have `b_i = x <= y <= t_i`.

  + Case `x = b_i - 1`.

    The value `x` should be read from `WF_j` for some `O_j` that is `pop()` taking the irregular
    path. Let `w` be the value `O_j` read from `top` at `'L204`. Since `O_j` takes the irregular
    path, we have `w >= x`.

    Let's prove that there is an invocation `I` that writes `x+1` to `top`. If `w >= x+1`, then the
    conclusion follows immediately. If `w = x`, then `O_j` tries to CAS `top` from `x` to `x+1`. It
    writes `x+1` to `top` unless if there is a concurrent `steal()` invocation that writes `x+1` to
    `top`.

    Now let's prove that `I` should be linearized before `I_i`. If `I` is `pop()`, then either `I =
    O_j` or `I` should be linearized before `O_j` by the coherence of `top`. Thus `I` is linearized
    before `I_i`. If `I` is `steal()`, then by a release-acquire synchronization from `I`'s
    release-store at `'L410` to `O_j`'s acquire-load at `'L213`, the value `I` read from `bottom`
    should be coherence-before `WL_j`. Thus `I` should be linearized before `O_j` and in turn `I_i`.

    Thus we have and `x+1 <= t_i`, and in turn `b_i = x+1 <= t_i`.

    <!-- [I] -->
    <!-- r t x -->
    <!-- ----- -->
    <!-- r b >=x+1 -->
    <!-- u t x x+1 -->

    <!-- [I_i] -->
    <!-- r t >=x -->
    <!-- ----- -->
    <!-- r b x -->

    <!-- O_j: -->
    <!-- r b x+1 -->
    <!-- w b x -->
    <!-- ------- -->
    <!-- r t ? -->
    <!-- w b x+1 -->



# Discussion

## Informal Proof

The proof described in this RFC is informal. We leave its formal verification in a proof assistant
as a future work.


## Comparison to the Current Implementation

A reader may notice that the pseudocode differs from [the current implementation in
Crossbeam][deque-current], which is faithfully implementing the deque described in [this
paper][chase-lev-weak]. More specifically, in the current implementation:

- `push()` issues a release fence and a relaxed write instead of a release write (`'L110`);
- `pop()` loads the buffer earlier;
- `pop()`'s CAS uses SeqCst/Relaxed orderings for success/failure cases instead of Release/Acquire
  orderings;
- `resize()` issues a release-swap to the old buffer with a new one, instead of just a release-write
  (`'L307`);
- `steal()` issues a acquire-load from `top` instead of a relaxed-load (`'L401`);
- `steal()`'s CAS uses SeqCst/Relaxed orderings for success/failure cases instead of Release/Relaxed
  orderings;
- `steal()`'s CAS is strong rather than weak (`'L410`).

The proposed implementation is more optimized in terms of orderings, except that the CAS in `pop()`
has a stronger ordering for the failure case. But in x86, Power, and ARM architectures,
`Release`/`Acquire` orderings for success/failure cases for a CAS is most likely more efficient than
`SeqCst`/`Relaxed` orderings.


## Reasoning Out-of-thin-air Behavior

The current implementation of Chase-Lev deque is not linearizable in C11 as-is, because of
out-of-thin-air behaviors. Here is an example. Suppose that the deque consists of a single value
`42`, with `bottom = 1` and `top = 0`. Two concurrent threads access the deque. The owner thread
calls `pop()`, and the stealer thread calls `steal()` twice. Consider the following execution:

```rust
// owner calls pop()
'L01: read(bottom, 1, Relaxed)
'L02: write(bottom, 0, Relaxed)
'L03: fence(SeqCst)
'L04: read(top, 1, Relaxed) // from `'L17`
'L05: write(bottom, 1, Relaxed)
// return `Empty`

||

// stealer calls steal()
'L11: read(top, 0, Relaxed)
'L12: fence(SeqCst)
'L13: read(bottom, 0, Relaxed) // from `'L02`
// return `Empty`

// stealer calls steal()
'L14: read(top, 0, Relaxed)
'L15: fence(SeqCst)
'L16: read(bottom, 1, Relaxed) // from `'L05`
'L17: CAS(top, 0, 1, Release) // success
// return `42`
```

The C11 allows this execution, but it really should not: there is no linearization of these method
invocations. There is a [proposal to ban out-of-thin-air behaviors][n3710] in the C11 standard, but
the proposal only vaguely described what does "out-of-thin-air" mean. You can roughly regard the
promising semantics as a precise definition of out-of-thin-air behaviors.

The promising semantics forbids this behavior because `'L05` has a control dependency on `'L04` and
`'L17` is a release-store so that there is a dependency cycle among `'L04`, `'L05`, `'L16`, and
`'L17`. See the proof of `(STEAL-IRREGULAR)` for more details.


## Efficiency

The orderings in the proposed implementation are Pareto-optimal: any lowering of any single ordering
breaks the linearizability.

In particular, the `Acquire` ordering at `'L213` is necessary. Without it, the following execution
is possible, but is not linearizable:

```rust
// owner calls pop()
'L01: read(bottom, 1, Relaxed)
'L02: write(bottom, 0, Relaxed)
'L03: fence(SeqCst)
'L04: read(top, 0, Relaxed)
'L05: CAS(top, 0, 1, Release) // fail; read from from `'L17`
'L06: write(bottom, 1, Relaxed)
// return `Empty`

||

// stealer calls steal()
'L11: read(top, 0, Relaxed)
'L12: fence(SeqCst)
'L13: read(bottom, 0, Relaxed) // from `'L02`
// return `Empty`

// stealer calls steal()
'L14: read(top, 0, Relaxed)
'L15: fence(SeqCst)
'L16: read(bottom, 1, Relaxed) // from `'L06`
'L17: CAS(top, 0, 1, Release) // success
// return `42`
```

Note that this example is similar to the one above, but there is no control dependency from `'L04`
to `'L06`. So in order to forbid this execution, we have to ensure the ordering of `'L05` and
`'L06`, by e.g. either (1) `Release`-write at `'L216`, (2) requiring `SeqCst` ordering for the
success case for the CAS at `'L213` (as done in [this paper][chase-lev-weak]), or (3) requiring
`Acquire` ordering for the failure case for the CAS at `'L213`. We chose (3) here, because we
believe it incurs the least performance overhead.

Even though `Release`/`Acquire` orderings are enough for the success/failure cases of the CAS at
`'L213`, C11 (and effectively also LLVM and Rust) requires that "the failure argument shall not be
memory\_order\_release nor memory\_order\_acq\_rel. The failure argument shall be no stronger than
the success argument." ([C11][c11] 7.17.7.4 paragraph 2). So we used `AcqRel`/`Acquire` in the
implementation. I think this C11 requirement may be failsafe for most use cases, but is actually
inefficient for the Chase-Lev deque.


## Comparison to Target-dependent Implementations

Alternatively, we can write a deque for each target architecture in order to achieve better
performance. For example, [this paper][deque-bounded-tso] presents a variant of Chase-Lev deque in
the "bounded TSO" x86 model, where you don't need to issue the expensive `MFENCE` barrier in `pop()`
(think: `SeqCst` fence). Also, [this paper][chase-lev-weak] presents a version of the Chase-Lev
deque for ARMv7 that doesn't issue an `isync`-like fence, while the proposed implementation issues
one. Probably `Consume` is relevant here. These further optimizations are left as future work.



[chase-lev]: https://pdfs.semanticscholar.org/3771/77bb82105c35e6e26ebad1698a20688473bd.pdf
[chase-lev-weak]: http://www.di.ens.fr/~zappa/readings/ppopp13.pdf
[fence-free-stealing]: http://www.cs.tau.ac.il/~mad/publications/asplos2014-ffwsq.pdf
[deque-current]: https://github.com/crossbeam-rs/crossbeam-deque
[deque-bounded-tso]: http://www.cs.tau.ac.il/~mad/publications/asplos2014-ffwsq.pdf
[deque-pr]: https://github.com/jeehoonkang/crossbeam-deque/tree/deque-proof
[promising]: http://sf.snu.ac.kr/promise-concurrency/
[promising-coq]: https://github.com/snu-sf/promising-coq
[synch-patterns]: https://jeehoonkang.github.io/2017/08/23/synchronization-patterns.html
[linearizability]: https://en.wikipedia.org/wiki/Linearizability
[cppatomic]: http://en.cppreference.com/w/cpp/atomic/atomic
[n3710]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3710.html
[c11]: www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf

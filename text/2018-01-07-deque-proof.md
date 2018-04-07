# Summary

This RFC proposes an implementation of concurrent work-stealing double-ended queue (deque) on
Crossbeam, and its correctness proof. this RFC has a [companion PR to `crossbeam-deque`][deque-pr].



# Motivation

The current implementation of deque is not fully optimized yet, leaving non-negligible room for
performance improvement. However, it is unclear how to further optimize the current implementation
while guaranteeing its correctness. This RFC resolves this issue by proving an optimized
implementation.



# Detailed design

In this section, we present a pseudocode of deque, and prove that (1) it is safe in the C/C++ memory
consistency model, and (2) it is functionally correct, i.e. it acts like a deque.


## Semantics of C/C++ relaxed-memory concurrency

For the semantics of C/C++ relaxed-memory concurrency on which the deque implementation is based, we
use the state-of-the-art [promising semantics][promising]. For a gentle introduction to the
semantics, we refer the reader to [this blog post][synch-patterns].

One caveat of the original promising semantics is its lack of support for memory allocation. In this
RFC, we use the following semantics for memory allocation:

- In the original promising semantics, a timestamp was a non-negative quotient number: `Timestamp =
  Q+`. Now we define `Timestamp = Option<Q+>`, where `None` means the location is unallocated.
  Suppose `None < Some(t)` for all `t`.

- At the beginning, a thread's view is `None` for all the locations. When a thread allocates a
  location, its view on the location becomes `0`.

- If a thread is reading a location and it's view on the location is `None`, i.e. it is unallocated
  in the thread's point of view, then the behavior is undefined.

We believe this definition is compatible with all the results presented in the [the promising
semantics paper][promising]. We hope [its Coq formalization][promising-coq] be updated accordingly
soon.


## Proposed implementation

A work-stealing deque is owned by the "owner" thread (`Deque<T>`), which can push items to and pop
items from the "bottom" end of the deque. Other threads, which we call "stealers" (`Stealer<T>`),
try to steal items from the "top" end of the deque. Due to CAS failure, a steal invocation may
return `Retry`, which basically means the invocation did nothing and you should retry. Deque's
interface is summarized as follows:

```rust
impl<T: Send> Deque<T> { // Send + !Sync
    pub fn new() -> Self { ... }
    pub fn stealer(&self) -> Stealer<T> { ... }

    pub fn push(&self, value: T) { ... }
    pub fn pop(&self) -> Option<T> { ... } // None if empty
}

impl<T: Send> Stealer<T> { // Send + Sync + Clone
    pub fn steal(&self) -> Steal<T> { ... }
}

enum Steal<T> {
    Data(T),
    Empty,
    Retry, // CAS failed
}
```

Chase and Lev proposed [an efficient implementation of deque][chase-lev] on sequentially consistent
memory model, and Lê et. al. proposed [its variant for ARM and C/C++ relaxed-memory
concurrency][chase-lev-weak]. This RFC presents an even more optimized variant for C/C++.

A Chase-Lev deque consists of (1) the `bottom` index for the owner, (2) the `top` index for both the
owner and the stealers, and (3) the underlying `buffer` that contains items. When the buffer is too
small or too large, the owner may replace it with one with more appropriate size:

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

The `Deque` and `Stealer` structs have a counted reference to the inner implementation, and delegate
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

The pseudocode for `Inner<T>`'s `fn push()`, `fn pop()`, and `fn steal()` is as follows:

```rust
pub fn push(&self, value: T) {
    'L101: let b = self.bottom.load(Relaxed);
    'L102: let t = self.top.load(Acquire);
    'L103: let mut buffer = self.buffer.load(Relaxed, epoch::unprotected());

    'L104: let cap = buffer.get_capacity();
    'L105: if b - t >= cap {
    'L106:     self.resize(cap * 2);
    'L107:     buffer = self.buffer.load(Relaxed, epoch::unprotected());
    'L108: }

    'L109: buffer.write(b % buffer.get_capacity(), value, Relaxed);
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
    'L211: let mut value = Some(buffer.read((b - 1) % buffer.get_capacity(), Relaxed));

    'L212: if len == 1 {
    'L213:     if self.top.compare_and_swap(t, t + 1, Release, Acquire).is_err() {
    'L214:         mem::forget(value.take());
    'L215:     }
    'L216:     self.bottom.store(b, Relaxed);
    'L217:     return value;
    'L218: }

    'L219: let cap = buffer.get_capacity();
    'L220: if len < cap / 4 {
    'L221:     self.resize(cap / 2);
    'L222: }
    'L223: value
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

pub fn steal(&self) -> Steal<T> {
    'L401: let mut t = self.top.load(Relaxed);

    'L402: let guard = epoch::pin_fence(); // epoch::pin(), but issue fence(SeqCst) even if it is re-entering

    'L403: let b = self.bottom.load(Acquire);
    'L404: if b - t <= 0 { return Empty; }

    'L405: let buffer = self.buffer.load(Acquire, &guard);
    'L406: let value = buffer.read(t % buffer.get_capacity(), Relaxed);

    'L407: if self.top.compare_and_swap_weak(t, t + 1, Release, Relaxed).is_err() {
    'L408:     mem::forget(value);
    'L409:     return Retry;
    'L410: }

    'L411: Data(value)
}
```


## Safety

We first prove that the proposed implementation is memory-safe, i.e. it does not invoke undefined
behavior due to illegal memory accesses. We discuss two possible causes of undefined behavior: data
race and invalid pointer dereference.


### Absence of Data Races

All shared objects, except for the contents of `buffer`, are atomically accessed. For the contents
of `buffer`, the stealers only atomically reads from the contents, and the owner only reads from and
atomically writes to the contents. Thus there are no data races that invoke undefined behavior.

In particular, the stealers should atomically read from the contents of `buffer` (`'L406`) because
`fn push(): 'L109` may concurrently writes to the same object. For example, the scheduler may
preempt a `steal()` invocation right after `'L402` so that `t` read in `'L401` may be arbitrarily
stale. Now, suppose that in a concurrent `push()` invocation, `b` equals to `t +
buffer.get_capacity()` and it is overwriting a value to `buffer`'s `(b % buffer.get_capacity())`-th
element. At the same time, the `steal()` invocation may wake up and read `buffer`'s `(t %
buffer.get_capacity())`-th element.

What's the cost of atomic access to the buffer? It is worth noting that the deque's item may be
arbitrarily large, and [atomic accesses to large objects are probably blocking][cppatomic] in the
C/C++11 standard library. But actually this can be non-blocking, e.g. by interpreting a relaxed
store to a large object as concurrent relaxed stores to each word of the object. So we assume the
accesses to the contents of the buffer are non-blocking, and they add no additional synchronization
cost than plain accesses.



### Absence of Invalid Pointer Dereference

<!-- By invalid pointer dereference we mean (1) null pointer dereference, (2) use-after-free, or (3)
free-after-free. -->

For pointer safety, it is sufficient to prove that every access to the contents of the buffer
pointed-to by `Inner`'s `self.buffer` is safe, since (1) the other shared objects, namely
`self.bottom`, `self.top`, and `self.buffer`, are just integers or pointers, and (2) accesses to
thread-local storage are straightforwardly safe. It is worth noting that while the deque is shared
among the owner and its stealers, only the owner is able to call `fn push()`, `fn pop()`, and `fn
resize()`, thereby modifying `self.bottom` and `self.buffer`.

After `self.buffer` is initialized, it is modified only by the owner in `fn resize(): 'L307`. Since
the contents inside a buffer is always accessed modulo the buffer's capacity (`'L109`, `'L211`,
`'L406`) and the buffer's size is always nonzero, there are no buffer overruns.

Thus it remains to prove that the buffer is not used after it has been freed. Thanks to Crossbeam,
we don't need to take care of all the details of memory reclamation; the [relaxed-memory
RFC][rfc-relaxed-memory] proves that as a user of Crossbeam, we only need to show:

- When a buffer is deferred to be dropped (`'L308`), there are no longer references to the buffer in
  the shared memory, and no concurrent mutators introduce a reference to the buffer in the memory
  again.

  You can easily check these properties because the owner basically "owns" the buffer: only the
  owner introduces/eliminates references to a buffer to/from the shared memory. Indeed, before a
  buffer is deferred to be dropped at `'L308`, the only reference to the buffer via `self.buffer` is
  removed at `'L307`.

- With an `unprotected()` guard, we should not defer to drop an object that is concurrently accessed
  by another thread, and should not access an object that is concurrently deferred to be dropped by
  another thread.

  It is indeed the case because (1) a buffer is deferred to be dropped only by the owner with a
  protected guard, and (2) stealers do not use an unprotected guard.



## Functional Correctness

A more interesting question is: does the implementation act like a deque?


### Specification: Linearizability and Synchronization

In order to answer the question, we first define the sequential specification of a deque. A *deque
state* is a triple `(t, b, A)`, where `t` is the top index, `b` is the bottom index, and `A` is the
array of the values within the indexes `[t, b)`. A `push()` invocation increases the bottom index
`b`, and push the input value at the end of `A`. A `pop()` invocation decreases the bottom index `b`
and pop the last value from `A`, if `t < b`; otherwise, touches nothing and returns `EMPTY`. (As a
special case, if `t+1 = b`, you can increase the top index `t` instead of decreasing the bottom
index `b`.) A `steal()` invocation increases the top index `t` and pop the first value from `A`, if
`t < b`; otherwise, touches nothing and returns `EMPTY`. By `S -(I,R)-> S'` we denote the
*transition relation* described above, from a deque state `S` into `S'` by the invocation `I`
returning `R`.

A shared object is **linearizable** as a deque if the following holds:

> Suppose that multiple threads invoked methods (i.e. `push()`, `pop()`, `steal()`) on the shared
> object in a single execution. For those invocations that is not `steal()` returning `RETRY`, there
> exists a permutation `I_0`, ..., `I_(n-1)` of them, which we call **"linearization order"**,
> denoted by `<L`, such that:
>
> - (VIEW) For all `x` and `y`, if `I_x <V I_y`, then `x <L y` holds.
>
> Furthermore, there exists a deque state `(t_i, b_i, A_i)` for each `i`, such that:
>
> - (SEQ) Initially, `(t_0, b_0, A_0) = (0, 0, [])` holds. Furthermore, for all `i`, `(t_i, b_i,
>   A_i) -(signature(I_i),ret(I_i))-> (t_(i+1), b_(i+1), A_(i+1))` holds. In other words, the
>   invocations `I_0`, ..., `I_(n-1)` return outputs *as if* the invocations are sequentially
>   (i.e. non-concurrently) executed to a deque in the linearization order.

Here, by `signature(I)` and `ret(I)` we mean the function call signature and the return value of
`I`. Also, by `I <V J` we mean `view_end(I) < view_beginning(J)`, and by `view_beginning(I)` and
`view_end(I)` we mean the thread `I`'s view at the beginning and the end of the invocation. A view
`v1` is less than or equal to another view `v2`, denoted by `v1 <= v2`, if for any location `l`,
`v1[l] <= v2[l]` holds; and `v1` is less than `v2`, denoted by `v1 < v2`, if (1) `v1 <= v2`, and (2)
there exists a location `l` such that `v1[l] < v2[l]`.


In addition to linearizability, we require that a matching pair of `push()` and `steal()`
synchronize like a matching pair of a release-store and an acquire-load:

> - (SYNC) If a value is `push()`ed by an invocation `I` and then `steal()`ed by an invocation `J`,
>   then `view_beginning(I) <= view_end(J)` holds.

Note that a similar property for a matching pair of `push()` and `pop()` is a corollary of
linearizability, as only the owner is able to call `push()` and `pop()` and the `push()` should be
called before the `pop()` in the "program order" (the execution order in a single thread).


#### View-based Definition of Linearizability

The conditions `(VIEW)` and `(SEQ)` are a generalization of the conditions `L2` and `L1` in [the
traditional definition of linearizability][linearizability] to relaxed-memory concurrency. What is
unique in `(VIEW)` is that while the definition of `<H` (which corresponds to `<V` in `L2` in the
traditional definition) relies on totally ordered time, the definition of `<V` identifies the time
with view so that it is no longer totally ordered.

But as a side effect, `<V` cannot properly distinguish invocations with the same view. For example,
suppose the client code looks like:

```rust
fn client() {
    i_1();
    i_2();
}
```

and `view_beginning(i_1) = view_end(i_1) = view_beginning(i_2) = view_end(i_2)`. By definition, `i_2
-> i_1` is a legit linearization order, yet *morally* `i_1` should be ordered before `i_2` because
it is so in the program order. However, we do not regard it as a problem: if `i_1` and `i_2` do not
change the view, they are read-only, and should not change the state of the shared object. Thus it
doesn't matter whether `i_1` is considered to be before or after `i_2`.

<!-- If we want to be pedantic, we may alternatively require every method to write something to a -->
<!-- thread-local storage, effectively disallowing that the views at the beginning and the end should -->
<!-- differ. We can use this storage only in the proof, and optimize out in the runtime. -->


### Construction of Linearization Order

Consider the invocations to the deque in an execution. For the owner's invocations, the program
order is the only possible linearization among them. Let's assume that `O_0`, ..., `O_(o-1)` are the
owner's invocations, sorted according to the program order. We construct a full linearization order
by inserting stealers' invocations into it.

Note that every owner's invocation writes to `bottom`, and every stealers' invocation reads from
`bottom` at `'L403`. Using `bottom`, we classify the stealers' invocations into "groups" `{G_i}` as
follows:

- For a stealer's invocation `S`, if `S` reads the initial value from `bottom`, then `S ∈
  G_(-1)`. Otherwise, let `O_i` be such an owner's invocation that `S` read from `bottom` a value
  written by` O_i`.

- If `S` returns `EMPTY`, then `S ∈ G_i`.

- Otherwise (i.e. `S` returns a value), let `j` be such an index that `O_(j+1)`, ..., `O_i` are
  `pop()` taking the "irregular" path (i.e. not entering `'L219`) and `O_j` is not. (`j = -1` if
  `O_0`, ..., `O_i` are all `pop()` taking the irregular path.) Then `S ∈ G_j`.

We will insert the invocations in `G_i` between `O_i` and `O_(i+1)`. Inside a group `G_i`, we give
the linearization order as follows:

- Let `STEAL^x` be the set of steal invocations that stole an element at the index `x`, and
  `STEAL_EMPTY` be the set of steal invocations that returns `EMPTY`. Let `STEAL` be `Union_x
  {STEAL^x}`.

- Within `G_i`, place `STEAL` invocations first, and then `STEAL_EMPTY` invocations.

- If there are multiple `STEAL` invocations in `G_i`, order `STEAL^x` before `STEAL^y` if `x < y`.

- If there are multiple `STEAL_EMPTY` invocations in `G_i`, order `I` before `J` if `I <V J`. (There
  is such a linearization because the `<V` relation is acyclic. Also, there can be more than one
  linearizations of `STEAL_EMPTY` invocations.)


### Auxiliary Definitions and Lemma

Let `WF_i` and `WL_i` be `O_i`'s first and last write to `bottom`.

- > (VIEW-LOC): If `view_beginning(I)[loc] < view_end(J)[loc]` for some `loc`, then it is not the
  case that `J <V I`.

  Since `view_beginning(I)[loc] < view_end(J)[loc]` holds, it is not the case that
  `view_end(J) < view_beginning(J)` holds.

Suppose that `O_i` is `pop()` taking the irregular path, and `x` be the value `O_i` read from
`bottom` at `'L201`.

- > (IRREGULAR-TOP): We have [timestamp of `top = x`] <= `view_end(O_i)[top]`.

  First of all, the notation "[timestamp of `top = x`]" is well-defined, since `top` is
  monotonically increasing by CAS at `'L213` or `'L407`, so that there can be only one write `top =
  x` for each value `x`. For this reason, in the rest of this RFC, we intentionally confuse the
  values of `top` with the timestamps of `top`.

  Let `y` be the value `O_i` read from `top` at `'L204`. Since `O_i` is taking the irregular path,
  we have `x-1 <= y`. If `x <= y`, then the conclusion trivially follows. If `y = x-1`, then `O_i`
  tries to compare-and-swap (CAS) `top` from `x-1` to `x` at `'L213`. Regardless of whether the CAS
  succeeds or fails, `O_i` reads or writes `top >= x`.

  It is worth nothing that for this lemma to hold, it is necessary for the CAS at `'L213` to be
  strong, i.e. the CAS does not spuriously fail.

- > (IRREGULAR-STEAL): Let `S` be a successful `steal()` that reads from `bottom` a value written by
  `O_i`, and `y` be the value `S` writes to `top` at `'L407`. Then `y < x` holds.

  If `S` read `bottom = x-1` written by `O_i` at `'L202`, then `y <= x-1` from the condition at
  `'L404`. Otherwise, `S` read `bottom = x` written by `O_i` at `'L207` or `'L216`. We already know
  `y <= x` from the condition at `'L404`, so suppose `y = x` and let's derive a contradiction.

  + Case `S` read `bottom = x` written at `'L207`.

    Then `O_i` read `top >= x` at `'L204`. In order for `S` to read `bottom = x` at `'L403` and
    `O_i` to read `top >= x` at `'L204` at the same time, either `S`'s write to `top` at `'L407`
    should be promised before reading `bottom` at `'L403`, or `O_i`'s write to `bottom` at `'L207`
    should be promised before reading `top` at `'L204`. But this is impossible. For the former,
    `S`'s write to `top` at `'L407` is a release-store. For the latter, in order for `O_i` to
    promise to write `bottom = x` at `'L207`, `O_i` should have read `top = x-1` at `'L204` and then
    execute the CAS at `'L213` in certification. But a certain future memory mandates the CAS to
    fail, acquiring an arbitrarily high view, after which it is impossible to fulfill the
    promise. In short, there is a semantic dependency from `O_i`'s read from `top` at `'L204` to
    `O_i`'s write to `bottom` at `'L207`.

    It is worth noting that in architectures, such as ARM and Power, usually it is enough to show
    that there is syntactic dependency from `O_i`'s read from `top` at `'L204` to `O_i`'s write to
    `bottom` at `'L207`. We have to reason semantic dependency because we are targeting C-like
    languages, where compilers may perform various optimizations.

  + Case `S` read `bottom = x` written at `'L216`.

    Then `O_i` executes the CAS at `'L213`. Since `S` writes `top = x`, it is impossible for the CAS
    to succeed. Since the CAS is strong, `O_i` should have read `top >= x`. Then there is a
    release-acquire synchronization from `S`'s write to `top` at `'L407` to `O_i`'s read from `top`
    at `'L213`. Thus it is impossible for `S` to read from `bottom` at `'L403` the value `x` written
    by `O_i` at `'L216`.


### Proof of `(VIEW)`

For `(VIEW)`, it is sufficient to prove that:

1. `(VIEW-OWNER-STEAL)`: for all `i <= j` and `S ∈ G_j`, it is not the case that `S <V O_i`.
2. `(VIEW-STEAL-OWNER)`: for all `i < j` and `S ∈ G_i`, it is not the case that `O_j <V S`.
3. `(VIEW-STEAL-INTER-GROUP)`: for all `i < j`, `S_i ∈ G_i` and `S_j ∈ G_j`, it is not the case that
   `S_j <V S_i`.
4. `(VIEW-STEAL-INTRA-GROUP)`: for all `i` and `S, S' ∈ G_i`, if `S` is placed before `S'`, then it
   is not the case that `S' <V S`.

(Note that what would be `(VIEW-OWNER-OWNER)` is obvious from the fact that `{O_i}` is sorted
according to the program order.)

#### Proof of `(VIEW-OWNER-STEAL)`

Since `O_i` wrote `WF_i`, we have `view_beginning(O_i)[bottom] < Timestamp(WF_i)`. Furthermore,
since the value `S` read from `bottom` is coherence-after-or `WF_j`, we have `Timestamp(WF_j) <=
view_end(S)[bottom]`. Thus `view_beginning(O_i)[bottom] < Timestamp(WF_i) <= Timestamp(WF_j) <=
view_end(S)[bottom]`, and by `(VIEW-LOC)`, it is not the case that `S <V O_i`.

#### Proof of `(VIEW-STEAL-OWNER)`

If `S` returns `EMPTY`, then `view_beginning(S)[bottom] <= Timestamp(WL_i) < Timestamp(WL_j) <=
view_end(O_j)[bottom]`, and by `(VIEW-LOC)`, the conclusion follows.

Now suppose `S` returns a value. Let `O_k` be such an owner invocation that the value `S` read from
`bottom` at `'L403` is written by `O_k`. Then by the construction of the linearization order, for
all `l ∈ (i, k]`, `O_l` is `pop()` taking the irregular path. If `k < j`, then
`view_beginning(S)[bottom] <= Timestamp(WL_k) < Timestamp(WL_j) <= view_end(O_j)[bottom]`, and by
`(VIEW-LOC)`, the conclusion follows.

Now suppose `j ∈ (i, k]`. Let `x` and `y` be the values `O_j` read from `bottom` at `'L201` and
`top` at `'L204`, respectively. Since `O_l` is `pop`() taking the irregular path for all `l ∈ (j,
k]`, `O_k` also read `x` from `bottom` at `'L201`. Then we have `x-1 <= y` by the fact that `O_j` is
a `pop()` taking the irregular path, `view_beginning(S)[top] < x-1` by `(IRREGULAR-STEAL)`, and `x
<= view_end(O_j)[top]` by `(IRREGULAR-TOP)`. Thus we have `view_beginning(S)[top] < x-1 < x <=
view_end(O_j)[top]`, and by `(VIEW-LOC)`, the conclusion follows.

#### Proof of `(VIEW-STEAL-INTER-GROUP)`

- Case 1: `O_(i+1)`, ..., `O_j` are all `pop()` taking the irregular path.

  Then `S_j` should have returned `EMPTY`. If `S_i` read from `WF_i` or `WL_i`, then
  `view_beginning(S_i)[bottom] <= Timestamp(WL_i) < Timestamp(WF_j) <= view_ending(S_j)[bottom]`,
  and by `(VIEW-LOC)`, the conclusion follows.

  Not suppose otherwise. Then `S_i` should have returned a value. Let `x` be the value `O_(i+1)`,
  ..., `O_j` read from `bottom` at `'L201`. We have `view_beginning(S_i)[top] < x-1` by
  `(IRREGULAR-STEAL)`, and `x-1 <= view_end(S_j)[top]` by `(IRREGULAR-TOP)` and the fact that `S_j`
  read `x-1` or `x` from `bottom` at `'L403`. By `(VIEW-LOC)`, the conclusion follows.

- Case 2: Otherwise.

  Let `O_k` be such an invocation that is not `pop()` taking the irregular path and `k ∈ (i,
  j]`. Then we have `view_beginning(S_i)[bottom] < Timestamp(WF_k) <= Timestamp(WF_j) <=
  view_ending(S_j)[bottom]`, and by `(VIEW-LOC)`, the conclusion follows.

#### Proof of `(VIEW-STEAL-INTRA-GROUP)`

- Case 1: `S, S' ∈ STEAL`.

  Suppose `S` read the value `y` from `top` at `'L401`, and `S'` read the value `w` from `top` at
  `'L401`. By the construction, `y < w`. By `(VIEW-LOC)`, the conclusion follows.

- Case 2: `S, S' ∈ STEAL_EMPTY`. Obvious from the construction.

- Case 3: `S ∈ STEAL` and `S' ∈ STEAL_EMPTY`.

  Let `b` be the value of `WL_i`; `x` and `x'` be the values `S` and `S'` read from `bottom` at
  `'L404`, respectively; and `y` and `y'` be the values `S` and `S'` read from `top` at `'L401`,
  respectively. Since `S ∈ STEAL`, we have `y < x` from the condition at `'L404`. Since `S' ∈
  STEAL_EMPTY`, we have `x' <= y'`.

  If `S` read from `bottom` a value written by `O_i`, then we have `y < x = b = x' <=
  y'`. Otherwise, then `S` read from `bottom` a value written by `O_j` for some `j > i`, and for all
  `k ∈ (i, j]`, `O_k` is `pop()` taking the irregular path. By `(IRREGULAR-STEAL)`, we have `y+1 <
  b`. By the definition of `push()` and `pop()`, we have `b-1 <= x'`, and thus `y <= b-2 < b-1 <= x'
  <= y'`. In either case, we have `y < y'`, and by `(VIEW-LOC)`, the conclusion follows.


## Proof of `(SEQ)` and `(SYNC)`

Let `I_0`, ..., `I_(n-1)` be the invocations sorted according to the constructed linearization
order. In addition to `(SEQ)` and `(SYNC)`, we will simultaneously prove the following conditions
with the same existentially quantified values of `t_i`, `b_i`, `A_i`:

> `(BOTTOM)`: If `I_i` is an owner invocation, then `b_i` equals to the value `I_i` read from
> `bottom` at `'L101` or `'L201`.
>
> `(TOP)`: In `I_0`, ..., `I_(i-1)`, there are `t_i` invocations that succeed in updating `top`. Let
> `T_0`, ..., `T_((t_i)-1)` be such invocations, sorted according to the order in `I`. Then `T_x`
> updates `top` from `x` to `x+1`.
>
> `(CONTENTS)`: for all `x ∈ [t_i, b_i)`, there exists a `push()` invocation into `x` in `I_0, ...,
> I_(i-1)`; and `A_i[x]` is the value inserted by the last such invocation.

We prove that `{I_i}` satisfies `(SEQ)`, `(SYNC)`, `(BOTTOM)`, `(TOP)`, and `(CONTENTS)` by
induction on `n`: suppose `I_0`, ..., `I_(i-1)` satisfies those conditions, and let's prove that
there exists `b_(i+1)`, `t_(i+1)`, and `A_(i+1)` such that `I_i` also satisfies those conditions. We
prove for each case of `I_i`.

- Case 1: `I_i` is `push()`.

  Quite obvious.

- Case 2: `I_i` is `pop()` taking the regular path.

  Let `j` be such an index that `I_i = O_j`. Let `x` and `y` be the values `I_i` read from `bottom`
  at `'L201` and `top` at `'L204`, respectively. By `(BOTTOM)`, we have `x = b_i`. Since `I_i` is
  taking the regular path, we have `y+2 <= x`. We also prove `t_i <= y+1`. Consider the invocation
  `I` that writes `y+2` to `top`, if exists. (Otherwise, by `(TOP)`, `t_i <= y+1` should hold.) If
  `I` is `pop()`, then `I` should be linearized after `I_i` thanks to the coherence of `top`; if `I`
  is `steal()`, then by the synchronization of the seqcst-fences from `I_i` to `I` via `top`, the
  value `I` read from `bottom` at `'L403` should be coherence-after-or `WL_j`. Thus `I` is
  linearized after `I_i`, and by `(TOP)`, `t_i <= y+1` holds.

  Then we have `t_i <= y+1 < y+2 <= x = b_i`, and it is legit to pop a value from the `bottom` end
  of the deque and decrease `bottom`.

  Now we choose `b_(i+1) = b_i - 1`, `t_(i+1) = t_i`, and `A_(i+1) = A_i`, i.e. `I_i` pops a value
  from the `bottom` end of the deque. It remains to prove that `I_i` returns the right value. Let
  `O_k` be the last `push()` operation in `I_0`, ..., `I_(i-1)` that pushed to the index `x-1` and
  writes `bottom = x`, and `v` be the value `O_k` pushed. Thanks to `(CONTENTS)`, it is sufficient
  to prove that `I_i` returns `v`.

  For all `l`, let `WB_l` be the value of `buffer` at the beginning of the invocation `O_l`. Also,
  for all `z`, let `WC_(l, z)` be the `z`-th contents of the buffer `WB_l` at the beginning of the
  invocation `O_l`. These are well-defined since the pointer `buffer` and the contents of the buffer
  are modified only by the owner.

  Let's prove by induction that for all `l ∈ (k, j]`, `WC_(l, (x-1) % size(WB_l)) = v`. Since `O_k`
  just pushed a value to the index `x-1`, it trivially holds for the base case `l = k+1`. Now
  suppose that it holds for `l = m` for some `m ∈ (k, j-1]` and prove that it holds for `l =
  m+1`. Since `O_k` is the last operation that writes `bottom = x` before `O_j` writes `bottom =
  x-1`, `O_m` is not a regular `pop()` that writes `bottom = x-1`. If `O_m` is resizing, then for
  the values `z` and `w` that `O_m` read from `bottom` at `'L301` and `top` at `'L302`,
  respectively, we have `z >= x` by the choice of `O_k` and `w <= y` by the coherence on
  `top`. Since `y < x`, `WC_(m, (x-1) % size(WB_m)) = v` is copied to `WC_(m+1, (x-1) %
  size(WB_(m+1)))`.

  Thus `I_i = O_j` returns `WC_(j, (x-1) % size(WB_j))`, which equals to `v`.

- Case 3: `I_i` is `pop()` taking the irregular path.

  Let `j` be such an index that `I_i = O_j`. Let `x` and `y` be the values `I_i` read from `bottom`
  at `'L201` and `top` at `'L204`, respectively. By `(BOTTOM)`, we have `x = b_i`. Since `I_i` takes
  the irregular path, we have `y >= x-1`.

  + Case `y >= x`, or [`y = x-1` and the CAS at `'L213` fails].

    If the former is the case, then `I_i` writes `bottom = x` at `'L208`. If the latter is the case,
    then `I_i` read from `top` a value `>= x` at `'L213`, and then writes `bottom = x` at
    `'L216`. In either case, it reads `top >= x` and then writes `bottom = x`.

    We prove `x <= t_i` as follows. Consider the invocation `I` that writes `x` to `top`. By
    `(TOP)`, it is sufficient to prove that `I` is linearized before `I_i`. If `I` is `pop()`, then
    it is so thanks to the coherence of `top`. Now suppose `I` is `steal()`. Let `O_k` be the first
    `push()` invocation in `O_(j+1)`, ..., `O_(o-1)`. (If no such invocation exists, let `k =
    o`. Also recall that `o` is the number of owner's method invocations.)  By the coherence of
    `top`, all of `O_j`, ..., `O_(k-1)` should read `top >= x` at `'L204`, and take the irregular
    path. Thus it is sufficient to prove that the value `I` read from `bottom` at `'L403` should be
    coherence-before `WF_k`. Suppose otherwise. Then there is a release-acquire synchronization from
    `WF_k` to `I`'s read from `bottom` at `'L403`, so `O_j`'s read from `top` at `'L204` or `'L213`
    is coherence-before `I`'s write to `top` at `'L407`. But this is impossible due to the coherence
    of `top`.

    Thus we have `b_i = x <= t_i`, and `I_i` goes to `'L207`, restores the original value of
    `bottom`, and returns `EMPTY`. We choose `b_(i+1) = b_i`, `t_(i+1) = t_i`, and `A_(i+1) = A_i`,
    i.e. the deque is empty.

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

  + Case `y = x-1` and the CAS at `'L213` succeeds.

    `I_i` updates `top` from `x-1` to `x` at `'L213` and writes `bottom = x` at `'L216`. Let's prove
    that `x-1 <= t_i`. Consider the invocation `I` that writes `x-1` to `top`. By `(TOP)`, it is
    sufficient to prove that `I` is linearized before `I_i`. If `I` is `pop()`, then it is so thanks
    to the coherence of `top`. Now suppose `I` is `steal()`. Let `O_k` be the first `push()`
    invocation in `O_(j+1)`, ..., `O_(o-1)`. (If no such invocation exists, let `k = o`.) By the
    coherence of `top`, all of `O_j`, ..., `O_(k-1)` should be `pop()` taking the irregular
    path. Thus it is sufficient to prove that the value `I` read from `bottom` at `'L403` should be
    coherence-before `WF_k`. Suppose otherwise. Then there is a release-acquire synchronization from
    `WF_k` to `I`'s read from `bottom` at `'L403`, so `O_j`'s update of `top` from `x-1` to `x` at
    `'L213` is coherence-before `I`'s write to `top` at `'L407`. But this is impossible due to the
    coherence of `top`.

    From `x-1 <= t_i` and the fact that `I_i` writes `x` to `top`, we have `t_i = x-1`, and thus
    `t_i = x-1 = (b_i)-1`. We choose `b_(i+1) = b_i`, `t_(i+1) = t_i + 1`, and `A_(i+1) = A_i`,
    i.e. `I_i` pops the value from the `bottom` end of the deque and increases `top`. It remains to
    prove that `I_i` returns the right value, which is so for roughly the same reason above.

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

    <!-- r t x-2 -->

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

  Let `x` and `y` be the values `I_i` read from `bottom` at `'L403` and `top` at `'L401`,
  respectively. Then either `x = b_i` or `x = (b_i)-1` holds by the construction of linearization
  order and the definition of `push()` and `pop()`. Since `I_i` returns a value, we have `y < x`.
  let `k` be such an index that the value `I_i` reads from `bottom` at `'L403` is written by `O_k`.

  Let's prove that `y <= t_i`. Consider the invocation `I` that writes `y` to `top`. By `(TOP)`, it
  is sufficient to prove that `I` is linearized before `I_i`. Suppose `I` is `pop()`. Let `j` be
  such an index that `I = O_j`. From the fact that `I` writes `y` to `top`, we know that `I` is
  `pop()` taking the irregular path. Since there is a release-acquire synchronization from `I`'s
  write to `top` at `'L213` to `I_i`'s read from `top` at `'L401-'L402`, we have `j <= k`. Since `y
  < x`, it is not the case that `O_j`, ..., `O_k` are all `pop()` taking the irregular path. Thus `I
  = O_j` should be linearized before `I_i`.  If `I` is `steal()`, then there is a release-acquire
  synchronization from `I`'s write to `top` at `'L407` to `I_i`'s read from `top` at
  `'L401-'L402`. Thus the value `I_i` read from `bottom` at `'L403` is coherence-after-or the value
  `I` read from `bottom` at `'L403`. Thus `I` is linearized before `I_i`.

  From `y <= t_i` and the fact that `I_i` writes `y+1` to `top`, we have `t_i = y`. Thus `t_i = y <=
  x-1 = (b_i)-1`, and it is legit to steal the value from the `top` end of the deque and increase
  `top`.


  Now we choose `b_(i+1) = b_i`, `t_(i+1) = t_i + 1`, and `A_(i+1) = A_i`, i.e. `I_i` steals a value
  from the `top` end of the deque. It remains to prove that `I_i` returns the right value.

  Since `x > y`, there exists a `push()` invocation that pushed to the index `y` and writes `bottom
  = y+1`.  Let `O_k` be the last such an invocation in `I_0`, ..., `I_(n-1)`, and `v` be the value
  `O_k` pushed. By `(CONTENTS)`, it is sufficient to prove that `O_k` is linearized before `I_i`,
  and `I_i` returns `v`.

  Let's first prove that for all regular `pop()` invocation `O_j` that writes `bottom = y` at
  `'L202`, `O_j` should be linearized before `I_i`. In order for `O_j` to enter the regular path,
  `O_j` should have read from `top` a value `<= y-1` at `'L204`. Then by the synchronization of the
  seqcst-fences from `O_j` to `I_i` via `top`, the value `I_i` read from `bottom` at `'L403` is
  coherence-after-or `WF_j`. Thus `J` is linearized before `I_i`.

  Now let's prove that `O_k` is linearized before `I_i`. By the construction of the linearization
  order, it is sufficient to prove that the value `I_i` read from `bottom` at `'L403` is
  coherence-after-or `WF_k`. It is obvious if `O_k` is the only such an invocation that writes
  `bottom = y+1` at `'L110`. Otherwise, let `O_l` be the last such another invocation. Then there
  should be a regular `pop()` invocation `O_m` such that `l < m < k`, and `O_m` writes `bottom = y`
  at `'L202`. Thus `O_m` is linearized before `I_i`, and the value `I_i` read from `bottom` is
  coherence-after-or `WL_m`. Since `I_i` should have read `bottom >= y+1`, the value `I_i` read from
  `bottom` at `'L403` should be coherence-after-or `WF_k`.

  Also, for all `l > k`, `b_l > y` should hold. This is because there are no regular `pop()`
  invocations that writes `bottom = y` after `O_k`.

  <!-- [I_i] -->
  <!-- read t y -->
  <!-- -------- -->
  <!-- read b x (>= y+1) -->
  <!-- update t y y+1 -->

  <!-- [O_k: push(n)] -->
  <!-- read b y -->
  <!-- read t w -->
  <!-- write b y+1 (release) -->

  <!-- [pop(n)] -->
  <!-- read b y+1 -->
  <!-- write b y -->
  <!-- ---------- -->
  <!-- read t f (<= y-1) -->

  Let `O_l` be the owner invocation that writes `WB_(l+1)` to `buffer` that is read by `I_i`. Let
  `z` and `w` be the values `O_k` read from `bottom` at `'L101` and `top` at `'L102`, respectively.
  Since `I_i` is linearized after `O_k`, there is a release-acquire synchronization from `O_k`'s
  write to `bottom` at `'L110` to `I_i`'s read from `bottom` at `'L403`. Thus we have `w <= y`
  because `I_i` reads `top = y` once more at `'L407`, `WB_(l+1)` is coherence-after-or `WB_(k+1)`,
  and the value `v` that `O_k` wrote to the buffer at `'L109` should be acknowledged by `I_i` at
  `'L406`. Also, we have `view_beginning(O_k) <= view_end(I_i)`, as required by `(SYNC)`.

  + Case `l <= k`.

    Then `WB_(l+1) = WB_(k+1)`, and `I_i` read the return value from `WB_(l+1)[y %
    size(WB_(l+1))]`. This value should be coherence-after-or the value `v` written by `O_k` at
    `'L109`. Let's think of `O_m` that overwrites to `WB_(l+1)[y % size(WB_(l+1))]` after `O_k`. (If
    such `O_m` doesn't exist, then `I_i` should have read `v` and the conclusion follows.) Since
    `O_m` is after `O_k` and it read `bottom > y` at `'L101`, `O_m` is a `push()` operation that
    read `top > y` at `'L102`. Then there is a release-acquire synchronization from `I_i`'s write to
    `top` at `'L407` to `O_m`'s read from `top` at `'L102` so that `I_i`'s read from `WB_(l+1)[y %
    size(WB_(l+1))]` at `'L406` happens before `O_m`'s write to the same location. Thus `I_i` should
    have read `v` at `'L406`.

  + Case `k < l`.

    Let's prove by induction that for all `m ∈ [k+1, l+1]`, `WC_(m, y % size(WB_m)) = v`. By
    assumption, it holds for `m = k+1`. Now suppose that it holds for `m = n` for some `n ∈ [k+1,
    l+1)` and let's prove that it holds for `m = n+1`. Let `f` and `g` be the values `O_n` read from
    `bottom` and `top`. Then we have `g <= w <= y < b_n = f`. Since `k < n`, `O_n` is not a regular
    `pop()` that writes `bottom = y`. If `O_n` is resizing, then since `g <= y < f`, `WC_(n, y %
    size(WB_n))` is copied to `WC_(n+1, y % size(WB_(n+1)))`. Thus we have `WC_(n+1, y %
    size(WB_(n+1))) = v`.

    By the release-acquire synchronization from `O_l`'s write to `buffer` at `'L307` to `I_i`'s read
    from `buffer` at `'L405`, the value `I_i` read from the buffer's content at `'L406` should be
    coherence-after-or the value `WC_(l+1, y % size(WB_(l+1))) = v` that `O_l` wrote at
    `'L305`. Similarly to the above case, `I_i`'s read happens before any overwrites to the same
    location. Thus `I_i` should have read `v` at `'L406`.

  <!-- We also prove `t_i <= y` as follows. Consider the invocation `I` that writes `y+2` to `top`. If -->
  <!-- `I` is `pop()` whose last write to `bottom` is `WL_I`, then `I` should have read `y+1` from `top` -->
  <!-- at `'L213`. By positive piggybacking synchronization from `I_i`'s release-store at `'L407` to -->
  <!-- `I`'s acquire-load at `'L213`, `x` should be a value coherence-before `WL_I`. `I_i` should be -->
  <!-- linearized before `I`, and `t_i <= y` holds. If `I` is `steal()`, by positive piggybacking -->
  <!-- synchronization from `I_i`'s release-store at `'L407` to `I`'s acquire-load at `'L401-'L402`, the -->
  <!-- value `I` read from `bottom` at `'L403` should be coherence-after-or `x`. Since `I_i` writes `y+1` -->
  <!-- to `top` and `I` writes `y+2` to `top`, `I_i` should be linearized before `I`, and `t_i <= y` -->
  <!-- holds. -->


- Case 5: `I_i` is `steal()` and it returns `EMPTY`.

  Let `x` and `y` be the values `I_i` read from `bottom` at `'L403` and `top` at `'L401`,
  respectively. Since `I_i` returns `EMPTY`, `x <= y` holds. Also, either `x = b_i` or `x = (b_i)-1`
  holds by the construction of linearization order and the definition of `push()` and `pop()`.

  It is sufficient to prove that `b_i <= t_i`, since then it will be legit to return `EMPTY` by
  choosing `b_(i+1) = b_i`, `t_(i+1) = t_i`, and `A_(i+1) = A_i`.

  + Case `x = b_i`.

    We prove `y <= t_i` as follows. Consider the invocation `I` that writes `y` to `top`. By
    `(TOP)`, it is sufficient to prove that `I` is linearized before `I_i`. Suppose `I` is
    `pop()`. Let `j` be such an index that `I = O_j`. Then there is a release-acquire
    synchronization from `I`'s write to `top` at `'L213` and `I_i` read from `top` at `'L401-'L402`,
    and `x` should be coherence-after-or `WF_j`. Thus `I` should be linearized before `I_i`. If `I`
    is `steal()`, then by a release-acquire synchronization from `I`'s write to `top` at `'L407` to
    `I_i`'s read from `top` at `'L401-'L402`, `x` should be coherence-after-or the value `I` read
    from `bottom` at `'L403`. Thus `I` should be linearized before `I_i`.

    Then we have `b_i = x <= y <= t_i`.

  + Case `x = (b_i)-1`.

    The value `x` should be read from `WF_j` for some `O_j` that is `pop()` taking the irregular
    path. Let `k` be such an index that `O_j = I_k`. Since `I_i` returns `EMPTY`, we have `k <
    i`. Let `w` be the value `O_j` read from `top` at `'L204`. Since `O_j` takes the irregular path,
    we have `w >= x`. By `(BOTTOM)`, we have `b_k = x+1`.

    If `O_j` returns `EMPTY`, then we have `x+1 = b_k <= t_k <= t_i`. If `O_j` returns a value, then
    by `(TOP)`, we have `x+1 = t_(k+1) <= t_i`. In either case, we have `b_i = x+1 <= t_i`.

    <!-- <\!-- [O_j] -\-> -->
    <!-- <\!-- r b x+1 -\-> -->
    <!-- <\!-- w b x -\-> -->
    <!-- <\!-- ------- -\-> -->
    <!-- <\!-- r t x+1 -\-> -->
    <!-- <\!-- w b x+1 -\-> -->

    <!-- <\!-- [I_i] -\-> -->
    <!-- <\!-- r t >= x -\-> -->
    <!-- <\!-- ------- -\-> -->
    <!-- <\!-- r b x -\-> -->

    <!-- [I] -->
    <!-- r t x -->
    <!-- ------- -->
    <!-- r b x+1 -->
    <!-- u t x x+1 -->

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
- `pop()`'s CAS uses seqcst/relaxed orderings for success/failure cases instead of release/acquire
  orderings;
- `resize()` issues a release-swap to the old buffer with a new one, instead of just a release-write
  (`'L307`);
- `steal()` issues a acquire-load from `top` instead of a relaxed-load (`'L401`);
- `steal()`'s CAS uses seqcst/relaxed orderings for success/failure cases instead of release/relaxed
  orderings;
- `steal()`'s CAS is strong rather than (possibly) weak (`'L407`).

The proposed implementation is more optimized in terms of orderings, except that the CAS in `pop()`
has a stronger ordering for the failure case. But in x86, Power, and ARM architectures,
release/acquire orderings for success/failure cases for a CAS is most likely more efficient than
seqcst/relaxed orderings.


## Reasoning Out-of-thin-air Behavior

The current implementation of Chase-Lev deque is not linearizable in C11 as-is, because of so-called
"out-of-thin-air behaviors". Here is an example that is allowed in C11, but is not
linearizable. Suppose that the deque consists of a single value `42`, with `bottom = 1` and `top =
0`; two concurrent threads access the deque; and the owner thread calls `pop()`, and a stealer
thread calls `steal()` twice. Consider the following execution:

```rust
// owner calls pop()
'L201: read(bottom, 1, Relaxed)
'L202: write(bottom, 0, Relaxed)
'L203: fence(SeqCst)
'L204: read(top, 1, Relaxed) // value from `'L407`
'L207: write(bottom, 1, Relaxed)
// return `Empty`

||

// stealer calls steal()
'L401: read(top, 0, Relaxed)
'L402: fence(SeqCst)
'L404: read(bottom, 0, Acquire) // value from `'L202`
// return `Empty`

// stealer calls steal()
'L401: read(top, 0, Relaxed)
'L402: fence(SeqCst)
'L404: read(bottom, 1, Acquire) // value from `'L207`
'L407: CAS(top, 0, 1, Release) // success
// return `42`
```

While this execution is allowed in C11, it is called out-of-thin-air because there is a genuine
dependency cycle among `'L204`, `'L207`, `'L404` in the second call to `steal()`, and `'L407`. There
is a [proposal to ban out-of-thin-air behaviors][n3710] in the C11 standards, but the proposal only
vaguely described what does "out-of-thin-air" mean.

You can roughly regard the promising semantics as a precise definition of out-of-thin-air
behaviors. Indeed, the promising semantics forbids this execution. See the proof of
`(IRREGULAR-STEAL)` for more details.


## Optimal Orderings

The orderings in the proposed implementation are Pareto-optimal: any lowering of any single ordering
breaks the linearizability.

In particular, the failure ordering of the CAS at `'L213` should be acquire. Without it, the
following execution is possible, but is not linearizable:

```rust
// owner calls pop()
'L201: read(bottom, 1, Relaxed)
'L202: write(bottom, 0, Relaxed)
'L203: fence(SeqCst)
'L204: read(top, 0, Relaxed)
'L213: CAS(top, 0, 1, Release) // fail; value from `'L407`
'L216: write(bottom, 1, Relaxed)
// return `Empty`

||

// stealer calls steal()
'L401: read(top, 0, Relaxed)
'L402: fence(SeqCst)
'L404: read(bottom, 0, Relaxed) // value from `'L202`
// return `Empty`

// stealer calls steal()
'L401: read(top, 0, Relaxed)
'L402: fence(SeqCst)
'L404: read(bottom, 1, Relaxed) // value from `'L216`
'L407: CAS(top, 0, 1, Release) // success
// return `42`
```

Note that this example is similar to the one above, but there is no dependency from `'L213` to
`'L216`. So in order to forbid this execution, we have to ensure the ordering of `'L213` and
`'L216`, e.g. by either (1) release-write at `'L216`, (2) requiring seqcst ordering for the success
case for the CAS at `'L213` (as done in [this paper][chase-lev-weak]), or (3) requiring acquire
ordering for the failure case for the CAS at `'L213`, or (4) issuing an acquire fence after the CAS
at `'L213` fails. We chose (3) in this RFC, because we believe it incurs the least performance
overhead.

Even though release/acquire orderings are enough for the success/failure cases of the CAS at
`'L213`, C11 (and effectively also LLVM and Rust) requires that "The failure argument shall be no
stronger than the success argument." ([C11][c11] 7.17.7.4 paragraph 2). So instead, we chose (4) in
the companion implementation. This C11 requirement may be fail-safe for most use cases, but can
actually be slightly inefficient in this case.

It is worth noting that the CAS at `'L213` should be strong. Otherwise, a similar execution to the
one above is possible, where the CAS at `'L213` reads `top = 0` and then spuriously fails.


## Comparison to Target-dependent Implementations

Alternatively, we can write a deque for each target architecture in order to achieve better
performance.

We believe the proposed implementation is the most efficient in the x86-TSO model. Though [this
paper][deque-bounded-tso] presents a variant of various deques in the "bounded x86-TSO" model, where
you don't need to issue the expensive `mfence` barrier (think: seqcst-fence) in `pop()`.

For ARM/POWER, you can further optimize the compilation result of the proposed implementation as
follows:

- `'L102` can be just plain load: `'L109` is the only synchronization target, and they have RW ctrl
  dependency.

- `'L405` can be just plain load: `'L406` is the only synchronization target, and they have RR addr
  dependency. In an ideal world, this synchronizing dependency should be expressible in C11 using
  the `Consume` ordering.

- `'L404` can be just plain load, but `isync/isb` should be inserted right before `'L405`: `'L405`'s
  read, `'L406`'s read, `'L407`'s read/write, and the end view of `steal()` in the successful case
  are the synchronization targets, and they have RR/RW ctrl+`isync/isb` dependency.

  We believe [this paper][chase-lev-weak] has a bug in their ARMv7 implementation of Chase-Lev
  deque. Roughly speaking, they used a plain load for `'L404`, and put ctrl+`isync/isb` right after
  `'L406`. But in that case, the reads at `'L405-'L406` can be reordered before `'L404`. See
  the [this tutorial][arm-power] §4.2 on [the MP+dmb+ctrl litmus test][mp+dmb+ctrl] for more
  details.



[chase-lev]: https://pdfs.semanticscholar.org/3771/77bb82105c35e6e26ebad1698a20688473bd.pdf
[chase-lev-weak]: http://www.di.ens.fr/~zappa/readings/ppopp13.pdf
[fence-free-stealing]: http://www.cs.tau.ac.il/~mad/publications/asplos2014-ffwsq.pdf
[deque-current]: https://github.com/crossbeam-rs/crossbeam-deque
[deque-bounded-tso]: http://www.cs.tau.ac.il/~mad/publications/asplos2014-ffwsq.pdf
[deque-pr]: https://github.com/jeehoonkang/crossbeam-deque/tree/deque-proof
[rfc-relaxed-memory]: https://github.com/crossbeam-rs/rfcs/blob/master/text/2017-07-23-relaxed-memory.md
[promising]: http://sf.snu.ac.kr/promise-concurrency/
[promising-coq]: https://github.com/snu-sf/promising-coq
[synch-patterns]: https://jeehoonkang.github.io/2017/08/23/synchronization-patterns.html
[linearizability]: https://dl.acm.org/citation.cfm?id=78972
[cppatomic]: http://en.cppreference.com/w/cpp/atomic/atomic
[n3710]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3710.html
[c11]: www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf
[mp+dmb+ctrl]: https://www.cl.cam.ac.uk/~pes20/arm-supplemental/arm033.html
[arm-power]: https://www.cl.cam.ac.uk/~pes20/ppc-supplemental/test7.pdf

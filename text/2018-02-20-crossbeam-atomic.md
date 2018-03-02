# Summary

Create a new crate `crossbeam-atomic`, which will host a collection of
atomic utilites that complement those in `std::sync::atomic`.

No code is proposed as a starting point for the crate - this time we'll simply start
with an empty crate.

# Motivation

The atomics in `std::sync::atomic` (some examples are `AtomicUsize` and `AtomicPtr`)
are not enough. They provide a low-level interface that is just a thin wrapper around
the LLVM atomics. The interface is okay for building data structures, but not
great for casual use - e.g. for simple exchange of data among threads or shared counters.

More specifically, some of the problems with `std::sync::atomic` are:

1. Types like `AtomicU8` and `AtomicI32` have been unstable for years.

2. There is no generic `Atomic<T>`. For example, in order to have an atomic `[u8; 3]`,
   one has to serialize the array into an `usize` and then store it into `AtomicUsize`.

3. `AtomicPtr` is too basic. It works with `*mut T`, but not with `Box<T>` or `Arc<T>`.

4. Memory orderings are annoying to specify and easy to get wrong.

### Atomic references

Rust currently doesn't have anything quite like
[`AtomicReference`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html)
in Java. Well, there is `AtomicPtr`, but it cannot hold smart pointers.
There's also [`pinboard`](https://docs.rs/pinboard/2.0.0/pinboard/), but
it comes with a few limitations: each read bears the cost of cloning and
`T` must be `Clone + 'static`.

The lack of `AtomicReference` has been extensively discussed in hniksic's
[blog post series](https://morestina.net/blog/784/exploring-lock-free-rust-3-crossbeam)
on lock-free programming in Rust. He describes how to build a simple reader-writer
data structure in Java and then an equivalent one in Rust. It turns out that the Rust
version is much more complicated, much slower, and requires writing unsafe code.
We should really do something to address this problem.

There is a recently submitted 
[post on the internals forum](https://internals.rust-lang.org/t/atomiccell-t-actually-a-box-with-cell-semantics-and-lock-free-atomicity/6699)
where the author is facing a similar problem. They also reached out for help in the
Rust IRC channels but didn't get a satisfactory answer. In the end, they simply gave
a shot at implementing this atomic primitive [themselves](https://bitbucket.org/SoniEx2/arccell-rs/src/0ef94e9bf3b6ee1647ebd2977baf8601372ae5eb/src/lib.rs?at=master&fileviewer=file-view-default).
The implementation looks okay, but is still going to be much slower than `AtomicReference`
in Java.

### Memory orderings

Let's see how other languages deal with memory orderings.
In [Java](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicInteger.html)
and [Go](https://golang.org/pkg/sync/atomic/),
all atomic operations are sequentially consistent - it is not possible to specify
weaker memory orderings at all!
In [C++](http://en.cppreference.com/w/cpp/atomic/atomic/load),
however, it is possible to specify a precise memory ordering for every operation,
but it is not necessary. If unspecified, the default ordering is always sequentially
consistent. Unfortunately, Rust doesn't have default values for function arguments so
we cannot emulate C++ here.

Note that having sequentially consistent ordering as the default one for all
atomic operations in C++ is a big win for ergonomics and correctness. One
can teach atomics by simply saying *"if you don't know what memory ordering are,
just don't worry about them"*. Moreover, if you only use sequentially consistent
orderings (or don't even specify them), then atomics become very intuitive and
they do exactly what one would expect - there can be no surprises
due to unexpected reorderings!

Similar reasoning was used for deciding that `std::shared_ptr`
in C++ must be thread-safe, despite the fact that the decision came with a non-negligible
performance cost. Fortunately, we have a stronger type system in Rust that allows
safe coexistence of both `Rc` and `Arc` without any footguns.
But atomics in Rust (`std::sync::atomic`) require users to specify an ordering
for absolutely every operation, which is a potential footgun.
Saying *"just use `SeqCst` everywhere"* doesn't seem to work - users
will still get orderings wrong. Some examples:

1. Julia Evans writes in a [blog post](https://jvns.ca/blog/2014/12/14/fun-with-threads/):
   *I’m not going to talk about the Relaxed right now (because I don’t understand it as well
   as I’d like), but basically this increments our counter in a threadsafe way (so that two
   threads can’t race).* <br>
   Choosing `Relaxed` can be risky - especially so if one doesn't understand
   it really well. She decided to pick `Relaxed` anyways, which means the
   language and documentation are failing at guiding users in writing correct code.

2. A [post](https://www.reddit.com/r/rust/comments/6y97qc/please_review_atomic_counter_thin_layer_around/)
   on Reddit: a Java programmer coming to Rust decides to wrap `AtomicUsize` into a simpler
   atomic counter that doesn't bother them about orderings on every operation (something
   akin to `AtomicInteger` in Java). They choose `Relaxed` as the default ordering.

3. A Rustacean unsatisfied with `AtomicPtr`
   [asks for](https://www.reddit.com/r/rust/comments/3v6ynx/shouldnt_there_be_an_atomicref/)
   `AtomicRef`. There's a lot of talk about scary `Relaxed` orderings. Reem suggests using
   the [`atomic-option`](https://github.com/reem/rust-atomic-option) crate, which is
   unsound in the same way `AtomicOption` in Crossbeam is.
   Atomics in Rust are a minefield. :)

Judging by these stories, atomics seem to be a mess for casual users. That doesn't
mean `std::sync::atomic` is broken - it only means we lack alternatives.

# Detailed design

Here are some quick ideas for atomic primitives we might want to add to the crate.
Keep in mind that these are not real proposals yet - we should think them through
more thoroughly in future RFCs.

### 1. `Atomic<T>`

The [`atomic`](https://docs.rs/atomic) crate generalizes atomics
in `std` into a single type `Atomic<T>`. The idea is that for small `T`s
we rely on an internal `AtomicUsize` and fall back to global spinlocks for big `T`s.
Implementations of `std::atomic<T>` in C++ work very similarly, and we should
have something like that in `crossbeam-atomic`.

Type `T` is not fully generic; only types that implement `Copy` are accepted.

Note that all operations require specifying a memory ordering, so this can
still be considered a low-level atomic primitive.

Here's a suggestion for the public API (very similar to the `atomic` crate):

```rust
struct Atomic<T: Copy> {
    // Transmuted to `AtomicUsize` when `size_of::<T>() != size_of::<usize>()`.
    cell: UnsafeCell<T>,
}

impl<T: Copy> Atomic<T> {
    fn new(val: T) -> Atomic<T>;
    fn get_mut(&mut self) -> &mut T;
    fn into_inner(self) -> T;

    // Returns `true` when `size_of::<T>() != size_of::<usize>()`.
    fn is_lock_free() -> bool;

    fn load(&self, order: Ordering) -> T;
    fn store(&self, val: T, order: Ordering);
    fn swap(&self, val: T, order: Ordering) -> T;

    fn compare_and_swap(&self, current: T, new: T, order: Ordering) -> T;

    fn compare_exchange(
        &self, 
        current: T, 
        new: T, 
        success: Ordering, 
        failure: Ordering
    ) -> Result<T, T>;

    pub fn compare_exchange_weak(
        &self, 
        current: T, 
        new: T, 
        success: Ordering, 
        failure: Ordering
    ) -> Result<T, T>;
}

impl Atomic<bool> {
    fn fetch_and(&self, val: bool, order: Ordering) -> bool;
    fn fetch_nand(&self, val: bool, order: Ordering) -> bool;
    fn fetch_or(&self, val: bool, order: Ordering) -> bool;
    fn fetch_xor(&self, val: bool, order: Ordering) -> bool;
}

// Repeat this impl for `u8`, `i16`, `u16`, `i32`, `u32`, `i64`, `u64`, `isize`, `usize`.
impl Atomic<i8> {
    fn fetch_add(&self, val: i8, order: Ordering) -> i8;
    fn fetch_sub(&self, val: i8, order: Ordering) -> i8;
    fn fetch_and(&self, val: i8, order: Ordering) -> i8;
    fn fetch_or(&self, val: i8, order: Ordering) -> i8;
    fn fetch_xor(&self, val: i8, order: Ordering) -> i8;
}
```

### 2. `AtomicCell<T>`

Compared to `Atomic<T>`, this primitive is a bit easier to use
(none of the methods need an ordering argument)
and more general (only a few methods require `T: Copy`).
All operations use the `SeqCst` ordering.

Note that `AtomicCell<Option<Box<T>>>` would be equivalent to the `AtomicOption<T>` we currently
have in Crossbeam (and by the way, our `AtomicOption<T>` has an unsound API). In fact,
`AtomicCell<T>` is strictly more powerful and more general than `AtomicOption<T>`.

The main idea behind `AtomicCell<T>` is that it behaves just like `Cell<T>`,
except it is thread-safe. Every Rustacean should feel at home with it - it really
doesn't have many surprises.

Since this is a cell, we use a slightly different language for operations:
`replace` rather than `swap`, `get` rather than `load`, `set` rather than `store`,
and so on.

`AtomicCell<T>` would be the closest equivalent of classes like
[`AtomicBoolean`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicBoolean.html)
and [`AtomicInteger`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicInteger.html)
in Java.

```rust
struct AtomicCell<T> {
    cell: UnsafeCell<T>,
}

impl<T> AtomicCell<T> {
    fn new(val: T) -> AtomicCell<T>;
    fn get_mut(&mut self) -> &mut T;
    fn into_inner(self) -> T

    // Returns `true` when `size_of::<T>() != size_of::<usize>()`.
    fn is_lock_free() -> bool;

    fn set(&self, val: T);
    fn replace(&self, val: T) -> T;
}

impl<T: Default> AtomicCell<T> {
    fn take(&self) -> T;
}

impl<T: Copy> AtomicCell<T> {
    fn get(&self) -> T;

    // We might want to restrict this method to primitive types rather than
    // accepting any `T: Copy`.
    fn compare_and_swap(&self, current: T, new: T) -> T;
}

// That's a lot of methods - maybe we should select just a subset of them.
impl AtomicCell<bool> {
    // Maybe these should be operator impls (e.g. `impl ops::Add<bool> for AtomicCell<bool>`).
    fn and(&self, val: bool);
    fn nand(&self, val: bool);
    fn or(&self, val: bool);
    fn xor(&self, val: bool);

    fn and_get(&self, val: bool) -> bool;
    fn nand_get(&self, val: bool) -> bool;
    fn or_get(&self, val: bool) -> bool;
    fn xor_get(&self, val: bool) -> bool;

    fn get_and(&self, val: bool) -> bool;
    fn get_nand(&self, val: bool) -> bool;
    fn get_or(&self, val: bool) -> bool;
    fn get_xor(&self, val: bool) -> bool;
}

// Repeat this for `u8`, `i16`, `u16`, `i32`, `u32`, `i64`, `u64`, `isize`, `usize`.
// That's a lot of methods - maybe we should select just a subset of them.
impl AtomicCell<i8> {
    // Maybe these should be operator impls (e.g. `impl ops::Add<i8> for AtomicCell<i8>`).
    fn add(&self, val: i8);
    fn sub(&self, val: i8);
    fn and(&self, val: i8);
    fn or(&self, val: i8);
    fn xor(&self, val: i8);

    fn add_get(&self, val: i8) -> i8;
    fn sub_get(&self, val: i8) -> i8;
    fn and_get(&self, val: i8) -> i8;
    fn or_get(&self, val: i8) -> i8;
    fn xor_get(&self, val: i8) -> i8;

    fn get_add(&self, val: i8) -> i8;
    fn get_sub(&self, val: i8) -> i8;
    fn get_and(&self, val: i8) -> i8;
    fn get_or(&self, val: i8) -> i8;
    fn get_xor(&self, val: i8) -> i8;
}
```

### 3. `AtomicRefCell<T>`

This is just `RefCell<T>` that implements `Send` and `Sync`.
It is essentially equivalent to `RwLock<T>`, except `borrow()` and `borrow_mut()` panic
if the value is already borrowed. It can also be faster than `RwLock<T>` under
certain circumstances.

Servo uses its own implementation of
[`AtomicRefCell<T>`](https://docs.rs/atomic_refcell/0.1.1/atomic_refcell/).
See [this pull request](https://github.com/servo/servo/pull/14828).

```rust
struct AtomicRefCell<T> {
    state: AtomicUsize,
    value: UnsafeCell<T>,
}

impl<T> AtomicRefCell<T> {
    fn new(val: T) -> AtomicRefCell<T>;

    fn into_inner(self) -> T;
    fn get_mut(&mut self) -> &mut T;
    fn replace(&self, val: T) -> T;

    fn borrow(&self) -> AtomicRef<'_, T>;
    fn borrow_mut(&self) -> AtomicRefMut<'_, T>;

    fn try_borrow(&self) -> Result<AtomicRef<'_, T>, BorrowError>;
    fn try_borrow_mut(&self) -> Result<AtomicRefMut<'_, T>, BorrowMutError>;
}

struct AtomicRef<'a, T: 'a> { ... }

impl<'a, T> Deref for AtomicRef<'a, T> {
    type Target = T;
    // ...
}

struct AtomicRefMut<'a, T: 'a> { ... }

impl<'a, T> Deref for AtomicRefMut<'a, T> {
    type Target = T;
    // ...
}

impl<'a, T> DerefMut for AtomicRefMut<'a, T> {
    // ...
}
```

### 4. `HazardCell<T>`

`HazardCell<T>` is similar to `AtomicCell<T>`, except it allows `get` even for
non-`Copy` types. However, this generalization comes at a cost because now
`T` is more restricted - only `T`s that implement `Pointer` are accepted.

Method `get` works even without `T: Copy` because it sets a hazard pointer in
a TLS slot in order to prevent the value from destruction while it is being used.
It is important to note that this hazard pointer mechanism
would not delay destruction of garbage in the same way epoch-based GC does.
We can destroy a garbage object as soon as it neither exists in the `HazardCell` nor
in any active `HazardGuard<T>`s.
See more [here](https://github.com/crossbeam-rs/rfcs/issues/25)
under *"Eager garbage collection using HP"*.

The details are a bit involved and there doesn't exist anything quite similar (AFAIK),
so I'd like to stop here and follow up on `HazardCell<T>` later. Consider this idea
very experimental. We still don't even have a proof of concept. :)

This primitive would be a good alternative to `ArcCell<T>` and the closest equivalent of
[`AtomicReference`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html)
in Java.

```rust
struct HazardCell<T: Pointer> {
    // `T` is just a pointer, so it is representable as a `usize`.
    inner: AtomicUsize,
    _marker: PhantomData<T>,
}

unsafe impl<T: Pointer> Send for HazardCell<T> {}
unsafe impl<T: Pointer> Sync for HazardCell<T> {}

// A `Pointer` is just a smart pointer represented as one word.
unsafe trait Pointer {
    fn into_raw(self) -> usize;
    unsafe fn from_raw(raw: usize) -> Self;
}
unsafe impl<T> Pointer for Box<T> { ... }
unsafe impl<T> Pointer for Option<Box<T>> { ... }
unsafe impl<T> Pointer for Arc<T> { ... }
unsafe impl<T> Pointer for Option<Arc<T>> { ... }
unsafe impl<T> Pointer for Weak<T> { ... }
unsafe impl<T> Pointer for Option<Weak<T>> { .. }

impl<T: Pointer> HazardCell<T> {
    fn new(val: T) -> HazardCell<T>;

    fn get_mut(&mut self) -> &mut T;
    fn into_inner(self) -> T;

    fn get(&self) -> HazardGuard<T>;
    fn set(&self, val: T);
    fn replace(&self, val: T) -> HazardGuard<T>;

    fn compare_and_set(
        &self,
        current: &T,
        new: T,
    ) -> Result<HazardGuard<T>, CasError<T>>;

    fn compare_exchange(
        &self,
        current: &mut HazardGuard<T>,
        new: T,
    ) -> Result<(), T>;
}

impl<T: Pointer + Default> HazardCell<T> {
    fn take(&self) -> HazardGuard<T>;
}

struct CasError<T> {
    current: HazardGuard<T>,
    new: T,
}

struct HazardGuard<T> { ... }

impl<T> Deref for HazardGuard<T> {
    type Target = T;
    // ...
}

impl<T> Drop for HazardGuard<T> {
    fn drop(&mut self) {
        // Releases the hazard pointer and destroys the object
        // if this is the last reference to it.
        // ...
    }
}
```

### 5. `EpochCell<T>`

The interface of `EpochCell<T>` looks almost identical to `HazardCell`,
except it requires `T: 'static`.
The bound is required because unlinked objects are handled by the epoch GC,
which means garbage could potentially be dropped very late in the future.

Generally speaking, you should use `HazardCell<T>` if:

- the cell is rarely updated
- you need eager destruction
- the allocated values are big/expensive

Use `EpochCell<T>` instead if:

- the cell is often updated
- lazy destruction is tolerable
- the allocated values are small/cheap

Other similar primitives:

* [`pinboard`](https://github.com/bossmc/pinboard)
* [`EbrCell`](https://github.com/Firstyear/crossbeam-concread/pull/1)
* [`rcu_ptr`](https://github.com/martong/rcu_ptr) (in C++)

```rust
// We're using the same `Pointer` trait from the `HazardCell` API.
struct EpochCell<T: Pointer + 'static> {
    // `T` is just a pointer, so it is representable as a `usize`.
    inner: AtomicUsize,
    _marker: PhantomData<T>,
}

unsafe impl<T: Pointer + 'static> Send for EpochCell<T> {}
unsafe impl<T: Pointer + 'static> Sync for EpochCell<T> {}

impl<T: Pointer + 'static> EpochCell<T> {
    fn new(val: T) -> EpochCell<T>;

    fn get_mut(&mut self) -> &mut T;
    fn into_inner(self) -> T;

    fn get(&self) -> EpochGuard<T>;
    fn set(&self, val: T);
    fn replace(&self, val: T) -> EpochGuard<T>;

    fn compare_and_set(
        &self,
        current: &T,
        new: T,
    ) -> Result<EpochGuard<T>, CasError<T>>;

    fn compare_exchange(
        &self,
        current: &mut EpochGuard<T>,
        new: T,
    ) -> Result<(), T>;
}

struct CasError<T> {
    current: EpochGuard<T>,
    new: T,
}

impl<T: Pointer + Default + 'static> EpochCell<T> {
    fn take(&self) -> EpochGuard<T>;
}

struct EpochGuard<T> {
    guard: crossbeam_epoch::Guard,
    // ...
}

impl<T> Deref for EpochGuard<T> {
    type Target = T;
    // ...
}
```

### 6. `Adder<T>` and `Accumulator<T>`

These primitives should be similar to classes like
[`LongAdder`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAdder.html)
and
[`DoubleAccumulator`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/DoubleAccumulator.html)
in Java. They are essentially just atomic numbers with internal sharding that
reduces the cost of contention.

I haven't thought too much about what the actual interface might look like in Rust, though.
We need to do some research on counting networks. Leaving this as an open question.
Help would be very welcome!

Note: crate [`counting-networks`](https://github.com/declanvk/counting-networks)
seems relevant here and might give us some inspiration.

# Drawbacks

N/A

# Alternatives

Perhaps we could cram these atomic primitives into `crossbeam-utils`, but there
will be plenty of them so it makes sense to dedicate a new crate.

Also, there are possible complications with adding atomics to `crossbeam-utils`:
`crossbeam-epoch` depends on `crossbeam-utils`, so if we were to add `EpochCell`
to `crossbeam-utils`, it would create a cyclic dependency on `crossbeam-epoch`.

# Unresolved questions

The main question is: which atomic primitives do we need and how exactly
should they look? That question should be addressed by future RFCs.

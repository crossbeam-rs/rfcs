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

1. The [`atomic`](https://docs.rs/atomic/0.3.4/atomic/) crate generalizes atomics
   in `std` into a single type `Atomic<T: Copy>`. The idea is that for small `T`s
   we use atomics from `std` and fall back to spinlocks for big `T`s.
   Implementations of `std::shared_ptr` in C++ work very similarly.
   Note that all operations require specifying a memory ordering, so this is
   still a low-level atomic primitive. We might want to move the crate into
   `crossbeam-atomic`.

2. `AtomicCell<T>` - a higher-level alternative to `Atomic<T>`. It doesn't require
   `T: Copy` and it is not possible to specify orderings. It should be possible
   to store `Box`es or `Arc`s inside it (e.g. `AtomicCell<Option<Arc<T>>>`).
   The best way of describing this primitive is: it works and is used almost
   exactly like `Cell<T>`, except it is thread-safe. Just like with `Cell`,
   `get` operation requires `T: Copy`, but `replace` doesn't.
   Also note that `AtomicCell<Option<Box<T>>>` is equivalent to `AtomicOption<T>`.
   Crate [`atomic_cell`](https://docs.rs/atomic_cell/0.1.0/atomic_cell/) has a
   somewhat similar interface, but in it all operations rely on locking.

3. `HazardCell<T: HazardPtr>` - Similar to `AtomicCell<T>`, except it allows `get`
   for some non-`Copy` types by using hazard pointers. The `get` operation returns
   a value of type `HazardCellGuard<'hc, T>`. Type `T` can be one of:
   `Box<_>`, `Option<Box<_>>`, `Arc<_>`, `Option<Arc<_>>`, `Weak<_>`, `Option<Weak<_>>`.
   Note that `HazardCell<Arc<T>>` is a good alternative to `ArcCell<T>`.
   This primitive should be the go-to answer for people asking for
   `AtomicReference` in Rust.

4. `EpochCell<T>` - Similar to [`pinboard`](https://github.com/bossmc/pinboard),
   [`EbrCell`](https://github.com/Firstyear/crossbeam-concread/pull/1), and
   [`rcu_ptr`](https://github.com/martong/rcu_ptr) (in C++).

5. `Adder<T>` and `Accumulator<T>` - similar to e.g.
   [`LongAdder`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAdder.html)
   and [`DoubleAccumulator`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/DoubleAccumulator.html)
   in Java. These are essentially just atomic numbers with internal sharding that
   reduces the cost of contention.
   Crate [`counting-networks`](https://github.com/declanvk/counting-networks)
   seems relevant here.

6. [`AtomicRefCell`](https://docs.rs/atomic_refcell/0.1.1/atomic_refcell/) is used by
   Servo. It might be a good idea to move it into Crossbeam.

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

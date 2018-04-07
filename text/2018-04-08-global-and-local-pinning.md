# Summary

Introduce the concept of *global pinning*, which allows one to pin the current thread without going through a thread-local handle.

In addition to that, we'll slightly tweak the general interface of `crossbeam-epoch` in order to fix several reported problems with it.

# Motivation

The following problems with the current `Collector`/`Handle`/`Guard` interface have been reported:

1. Users might mistakenly expect `Handle::clone()` to register a brand-new handle rather than returning a reference to the same handle. The method can be dangerously misleading.

2. Handles are not `Send`. This is an annoying problem and it's not obvious how to cheat around it correctly when you really have to.

3. Handles are not `Sync`. Sometimes it'd be nice to call `pin()` without registering a handle for each thread. In `SkipMap`, we need to pin inside `self.collector`, but creating a handle per-thread *and* per-instance is difficult. In `elfmalloc` there are situations where having to register handles is a nuisance.

4. `epoch::pin()` will panic if the thread-local handle is destroyed. Fortunately, `LocalKey::try_with` will be soon stabilized in Rust 1.26. But how do we work around the panics then - what should we do exactly in case the handle is destroyed?

# Detailed design

### Global pinning

Let's introduce something I've been referring to as *anonymous pinning* in the past, and has been briefly explained in [this comment](https://github.com/crossbeam-rs/crossbeam-epoch/pull/21#discussion_r144353198).

The idea is to have two atomic counters in the `Collector` (more concretely, in the `Global` struct). They count how many `Guards` are currently pinned in the global epoch and the epoch preceding it:

```rust
pin_counters: [AtomicUsize; 2],
```

We introduce a new method `Collector::pin()`, which is just like `Handle::pin()`:

```rust
impl Collector {
    fn pin(&self) -> Guard;
}
```

We'll have to distinguish between guards pinned globally and locally. Since we're turning `Guard` into an `enum`, let's also add a special case for the unprotected guard:

```rust
enum Guard {
    Global(*const Global),
    Local(*const Local),
    Unprotected,
}
```

Now, the `Global::pin()` method might be implemented like this:

```rust
fn pin(&self) -> Guard {
    // 0 if the epoch is even, or 1 if the epoch is odd.
    let parity = self.epoch.load(Ordering::Relaxed).parity();

    self.pin_counters[parity].fetch_add(1, Ordering::Relaxed);
    atomic::fence(Ordering::SeqCst);

    Guard::Global(self)
}
```

Of course, unpinning will simply decrement the counter.

**@jeehoonkang** [raised a concern](https://github.com/crossbeam-rs/crossbeam-epoch/pull/21#discussion_r144719080) with correctness of this method of pinning.

Consider this. What happens if we read a stale value of `self.epoch` and increment a counter just after the epoch advanced one step forward, and execute the fence just before the epoch is advanced one step forward again? That means we're effectively two epochs behind, while other threads might think we're zero epochs behind.

I think the problem is easily fixable by changing when a bag becomes expired. Rather than defining a bag is expired if it's at least 2 epochs old, we should say it's expired if it's at least 3 epochs old. That's it.

Finally, an important insight ought to be made about the difference between global (through `Collector`) and local (through `Handle`) pinning. Think of `Collector::pin()` as the *simple & easy* way of using the GC, and consider `Handle::pin()` just an optimized variant that avoids contention by writing to a special thread-local structure (a handle). In that sense, handles are a completely optional optimization, nothing more!

### Adjustments to handles

Let's remove `Handle::clone()` because it's misleading. It hasn't proved to be very useful so far, and is only used by `epoch::default_handle()`. Moreover, `epoch::default_handle()` hasn't been useful either so let's remove it, too.

On the other hand, `epoch::default_collector()` really is useful, and is used by `SkipMap` to initialize `self.collector` (of type `Collector`) to the default collector.

The omission of `epoch::default_handle()` isn't a big problem since we already have `epoch::pin()` and `epoch::flush()`. Note that `epoch::pin()` is a much better option than `epoch::default_handle().pin()` anyway - it can automatically fall back to global pinning if the thread-local handle is destroyed:

```rust
fn pin() {
    HANDLE.try_with(|h| h.pin()).unwrap_or_else(|| COLLECTOR.pin())
}
```

Next, we implement `Send` for `Handle` and mark `Handle::pin()` as unsafe:

```rust
unsafe impl Send for Handle {}

impl Handle {
    unsafe fn pin(&self) -> Guard;
}
```

The caller of `Handle::pin()` must pinky promise that the handle will not be sent to another thread while the `Guard` (or its clones) is alive. Unless, of course, the guard (and its clones) is simultaneously sent to the same thread, too.

### Remove `is_pinned()`

Another small tweak we'll do is remove `epoch::is_pinned()` and `Handle::is_pinned()`. These functions are only used by `crossbeam-deque` to make sure a fence is executed even if the thread is already pinned.

Instead, we'll add functions `epoch::pin_fence()`, `Handle::pin_fence()`, and `Collector::pin_fence()`, which are equivalent to `pin()`, except they guarantee to execute a `SeqCst` fence.

# Drawbacks

* `epoch::default_handle()` is gone. But it doesn't seem to be useful and we have `epoch::pin()` and `epoch::flush()` anyway. We can reintroduce it in the future if a need actually comes up.

* `Handle::clone()` is gone. However, nobody is using it and when it is used, it's used mistakenly. In short, it comes with practically zero benefits and a serious drawback.

* `Handle::pin()` is now unsafe. See the *Alternatives* section for what we can do about it.

# Alternatives

### Add safe `Guard::pin_with()` and remove `Guard::clone()`

We can add a safe alternative to `Guard::pin()`:

```rust
impl Handle {
    fn pin_with<F, R>(&self, f: F) -> R
    where
        F: FnOnce(&mut Guard) -> R;
}
```

This function is safe because it borrows the handle for the duration of pinning, which means it cannot be sent to another thread in the meantime. However, the user mustn't be able to clone the `Guard` and send the handle to another thread afterwards, as in this example:

```rust
let guard = handle.pin_with(|guard| guard.clone());
thread::spawn(move || {
    handle.pin_with(|_| ());
});
drop(guard); // Oops, the handle now belongs to another thread!
```

Therefore, we'd have to remove the `Clone` impl for `Guard`.

### Make `Handle::pin()` safe, but add a lifetime to `Guard`

Another alternative is to add a lifetime to `Guard` so that it becomes `Guard<'a>`. The lifetime `'a` borrows the `Handle` and therefore prevents it from being moved at all, which means it cannot be sent to another thread while such a guard exists.

This way we don't have to remove `Handle::clone()`.

Note that `Collector::pin()` and `epoch::pin()` simply return a `Guard<'static>`:

```rust
impl Collector {
    // We're not using a `Handle` - there's nothing to borrow, so we can use `'static`.
    fn pin(&self) -> Guard<'static>;
}

impl Handle {
    // The returned guard borrows `self`, preventing it from being sent to another thread.
    fn pin<'a>(&'a self) -> Guard<'a>;
}

// This function uses the default thread-local handle, but it cannot be sent to another thread,
// so we don't even have to borrow it.
fn pin() -> Handle<'static>;
```

The main drawback of this solution is that the lifetime might be at times annoying, but note that the lifetime can be simply elided in most real-world situations.

### Two kinds of handles: `SharedHandle` and `LocalHandle`

We could generalize the concept of global pinning and have a type of handle that can be shared between threads. It would work just like global pinning and holding two counters:

```rust
struct SharedHandle { // Send + Sync
shared: *const internal::Shared,
}

mod internal {
    struct Shared {
        // ...
        pin_counters: [AtomicUsize; 2],
    }
}

struct LocalHandle { ... } // Send

impl Collector {
    fn register_shared(&self) -> SharedHandle;
    fn register_local(&self) -> LocalHandle;
}

impl SharedHandle {
    fn pin(&self) -> Guard;
}

impl LocalHandle {
    unsafe fn pin(&self) -> Guard;
}
```

Note that now we don't need to have `Collector::pin()` and `Collector::flush()` because one can just create a `SharedHandle` and use it for global pinning.

This approach has one more important benefit. Consider how we might use global pinning in `SkipMap`:

```rust
struct SkipMap<K, V> {
    // ...
  
    collector: Collector,
}

impl SkipMap<K, V> {
    fn pin(&self) -> Guard {
        self.collector.pin()
    }

    fn insert(&self, k: K, v: V) -> Entry<K, V> {
        let guard = self.pin();
        // ...
    }
}
```

Updating the same set of counters in `self.collector.pin()` will cause contention and slow things down. How much it actually affects performance needs to be measured, but it is a concern.

A possible optimization is to create an array of distinct shared handles and choose one depending on the current thread, thus reducing the unwanted effects of contention:

```rust
struct SkipMap<K, V> {
    // ...
  
    collector: Collector,
    handles: [SharedHandle; 16],
}

impl SkipMap<K, V> {
    fn pin(&self) -> Guard {
        let id = thread_id();
        self.collector[id % 16].pin()
    }

    fn insert(&self, k: K, v: V) -> Entry<K, V> {
        let guard = self.pin();
        // ...
    }
}
```

# Unresolved questions

None so far.

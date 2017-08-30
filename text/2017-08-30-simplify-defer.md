# Summary

Simplify deferred destruction by removing `Scope::defer_free` and `Scope::defer_drop`.

# Motivation

In the [rsdb coco trip report](https://github.com/stjepang/coco/issues/7), **@spacejam**
reported a few papercuts with deferred destruction, which got me thinking about the interface
that is provided by Crossbeam:

```rust
impl Scope {
    // Deferred deallocation of heap-allocated object `ptr`.
    unsafe fn defer_free<T>(&self, ptr: Ptr<T>);

    // Deferred destruction and deallocation of heap-allocated object `ptr`.
    unsafe fn defer_drop<T: Send + 'static>(&self, ptr: Ptr<T>);

    // Deferred execution of arbitrary function `f`.
    unsafe fn defer<F: FnOnce() + Send + 'static>(&self, f: F);

    // ...
}
```

We have *three* very similar functions, which makes things a bit complicated. Can we simplify?

### Do we need `defer_free`?

This function is most useful when we need to simply deallocate nodes without running
destructors. For example, this is the layout of a node in a Treiber stack:

```rust
struct Node<T> {
    value: T,
    next: Atomic<Node<T>>,
}
```

And this is the part in `Stack::pop()` that deallocates nodes:

```rust
match self.head.compare_and_set_weak(head, next, AcqRel, scope) {
    Ok(()) => unsafe {
        scope.defer_free(head);
        return Some(ptr::read(&h.value));
    },
    Err(h) => head = h,
}
```

[`ManuallyDrop`](https://doc.rust-lang.org/stable/std/mem/union.ManuallyDrop.html) was
recently stabilized, which allows us to define nodes like this:

```rust
struct Node<T> {
    value: ManuallyDrop<T>,
    next: Atomic<Node<T>>,
}
```

Now we can easily avoid destructors without jumping through hoops:

```rust
match self.head.compare_and_set_weak(head, next, AcqRel, scope) {
    Ok(()) => unsafe {
        scope.defer_drop(head);
        return Some(ptr::read(&h.value));
    },
    Err(h) => head = h,
}
```

Neither `Atomic` nor `ManuallyDrop` have a destructor, so dropping a `Node` will be a noop,
which makes `defer_drop` in this case equivalent to `defer_free`.
This makes `defer_free` unnecessary.
This is the kind of problem `ManuallyDrop` was created to solve.

### Do we need `defer_free`?

Let's take a step further and rewrite the same piece of code using `defer`:

```rust
match self.head.compare_and_set_weak(head, next, AcqRel, scope) {
    Ok(()) => unsafe {
        let raw = head.as_raw() as usize;
        scope.defer(|| drop(Box::from_raw(raw as *const Node<T>)));
        return Some(ptr::read(&h.value));
    },
    Err(h) => head = h,
}
```

Calling the destructor will now come with some performance cost due to additional boxing
(`defer` stores `FnOnce()` in a `Box`).
Moreover, deferring destruction this way is a bit more unergonomic.
Let's fix both problems...

# Detailed design

### `Deferred`

Instead of having the triplet `defer_free`/`defer_drop`/`defer`, we'll have only `defer`
with the same interface:

```rust
impl Scope {
    // Deferred execution of arbitrary function `f`.
    unsafe fn defer<F: FnOnce() + Send + 'static>(&self, f: F);

    // ...
}
```

In most cases `defer` will be used to just deallocate memory or execute simple destruction
routines. The passed `FnOnce()` closure will almost always fit into, say, 3 words.

I have an [implementation](https://gist.github.com/stjepang/00d63b8febc07b297eae3480e80d0e91)
of `Deferred` ready, which is able to store a small closure within itself, falling back
to heap allocation for big closures.

**TL;DR:** `Deferred` is to `FnOnce() + Send + 'static` what `SmallVec` is to `Vec`.

### `Owned::from_ptr`

Finally, let's reconsider the piece of code from the motivation section and make it more ergonomic.
There are several unergonomic obstacles to destruction of `head`:

1. A `Ptr<'scope>` cannot be passed to a `'static` closure.
2. A raw pointer cannot be passed to a `Send` closure (a workaround is to convert the pointer to `usize`).
3. Finally, the raw pointer must be passed to `Box::from_raw`.

To simplify matters, let's introduce a new unsafe constructor, `Owned::from_ptr`:

```rust
match self.head.compare_and_set_weak(head, next, AcqRel, scope) {
    Ok(()) => unsafe {
        let head = Owned::from_ptr(head);
        scope.defer(move || drop(head));
        return Some(ptr::read(&h.value));
    },
    Err(h) => head = h,
}
```

Strictly speaking, this code is possibly incorrect because an `Owned` and `Ptr` pointing to the
same thing exist at the same time. This is okay because the `Owned` will not be used until the
closure executes, but **@RalfJung**'s new unsafe code checker might reject the code as unsound.

### `Ptr::to_static`

We can fix the problem by introducing another unsafe helper method, `Ptr::to_static`, which
converts `Ptr<'scope>` to `Ptr<'static>`:

```rust
match self.head.compare_and_set_weak(head, next, AcqRel, scope) {
    Ok(()) => unsafe {
        let head = head.to_static();
        scope.defer(|| drop(Owned::from_ptr(head)));
        return Some(ptr::read(&h.value));
    },
    Err(h) => head = h,
}
```

# Drawbacks

We'll introduce two new functions (`Owned::from_ptr` and `Ptr::to_static`). But at the same time
we'll also remove two functions (`Scope::defer_free` and `Scope::defer_drop`).

There might be very small differences in performance when using `defer_free` and `defer_drop` versus
`defer`, but most probably there won't be any or will be negligibly small in comparison to the cost
of `free`.

# Alternatives

An alternative could be to optimize `defer` by using `Deferred` instead of `SendBoxFnOnce`, and
at the same time keep `defer_free` and `defer_drop`.

# Unresolved questions

Bikeshedding new function names.

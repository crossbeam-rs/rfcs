# Summary

Simplify deferred destruction by removing `Scope::defer_free`.

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

# Can we make `defer` safe and faster?

`defer` is the most general function of the three:

```rust
unsafe fn defer<F: FnOnce() + Send + 'static>(&self, f: F);
```

There is no good reason for `defer` to be unsafe, which was an oversight when designing
the `Scope` API. So let's just make it safe.

Another issue with it is that the current implementation simply boxes the closure `f`.
But in most cases `defer` will be used to merely deallocate an array or execute a simple
destruction routine, so the passed closure will often fit into, say, 3 words.

To improve performance, we'll avoid allocation when possible by using something akin
to `SmallVec` for closures.

# Detailed design

Instead of having the triplet of unsafe functions `defer_free`/`defer_drop`/`defer`,
we'll have only unsafe `defer_drop` and safe `defer`:

```rust
impl Scope {
    // Deferred destruction and deallocation of heap-allocated object `ptr`.
    unsafe fn defer_drop<T: Send + 'static>(&self, ptr: Ptr<T>);

    // Deferred execution of arbitrary function `f`.
    fn defer<F: FnOnce() + Send + 'static>(&self, f: F);

    // ...
}
```

Regarding `defer` optimization, I have an implementation
of `Deferred` ready, which is able to store a small closure within itself, falling back
to heap allocation for big closures.

Put differently, `Deferred` is to `FnOnce() + Send + 'static` what `SmallVec` is to `Vec`.

`Deferred` is purely an implementation detail -- it does not appear in the public API.

<details>
<summary>Code (click to expand)</summary>
```rust
use std::mem;
use std::ptr;

/// Provides methods to dispatch a call to a `FnOnce()` from a trait object.
trait Callback {
    /// Calls the function from a trait object on the stack.
    ///
    /// This will copy `self`, call the function, and finally drop the copy.
    /// This method may be called only once, and `self` must not be dropped after that (tip: pass
    /// it to `std::mem::forget`).
    unsafe fn copy_and_call(&self);

    /// Calls the function from a trait object on the heap.
    fn call_box(self: Box<Self>);
}

impl<F: FnOnce() + Send + 'static> Callback for F {
    #[inline]
    unsafe fn copy_and_call(&self) {
        let f: Self = ptr::read(self);
        f();
    }

    #[inline]
    fn call_box(self: Box<Self>) {
        let f: Self = *self;
        f();
    }
}

/// The representation of a trait object like `&SomeTrait`.
///
/// This struct has the same layout as types like `&SomeTrait` and `Box<AnotherTrait>`.
///
/// It is actually already provided as `std::raw::TraitObject` gated under the nightly `raw`
/// feature. But we don't use nightly Rust, so the struct was simply copied over into Crossbeam.
///
/// If the layout of this struct changes in the future, Crossbeam will break, but that is a fairly
/// unlikely scenario.
// FIXME(stjepang): When feature `raw` gets stabilized, use `std::raw::TraitObject` instead.
#[repr(C)]
#[derive(Copy, Clone)]
struct TraitObject {
    data: *mut (),
    vtable: *mut (),
}

/// Some space to keep a `FnOnce()` object on the stack.
type Data = [u64; 4];

/// A small `FnOnce()` stored inline on the stack.
struct InlineObject {
    data: Data,
    vtable: *mut (),
}

/// A `FnOnce()` that is stored inline if small, or otherwise boxed on the heap.
///
/// This is a handy way of keeping an unsized `FnOnce()` within a sized structure.
enum Deferred {
    OnStack(InlineObject),
    OnHeap(Option<Box<Callback>>),
}

impl Deferred {
    /// Constructs a new `Deferred` from a `FnOnce()`.
    fn new<F: FnOnce() + Send + 'static>(f: F) -> Self {
        let size = mem::size_of::<F>();
        let align = mem::align_of::<F>();

        if size <= mem::size_of::<Data>() && align <= mem::align_of::<Data>() {
            unsafe {
                let vtable = {
                    let callback: &Callback = &f;
                    let obj: TraitObject = mem::transmute(callback);
                    obj.vtable
                };

                let mut data = Data::default();
                ptr::copy_nonoverlapping(
                    &f as *const F as *const u8,
                    &mut data as *mut Data as *mut u8,
                    size,
                );
                mem::forget(f);

                Deferred::OnStack(InlineObject { data, vtable })
            }
        } else {
            Deferred::OnHeap(Some(Box::new(f)))
        }
    }

    /// Calls the function or panics if it was already called.
    #[inline]
    fn call(&mut self) {
        match *self {
            Deferred::OnStack(ref mut obj) => {
                let vtable = mem::replace(&mut obj.vtable, ptr::null_mut());
                assert!(!vtable.is_null(), "cannot call `FnOnce` more than once");

                unsafe {
                    let data = &mut obj.data as *mut _ as *mut ();
                    let obj = TraitObject { data, vtable };
                    let callback: &Callback = mem::transmute(obj);
                    callback.copy_and_call();
                }
            }
            Deferred::OnHeap(ref mut opt) => {
                let boxed = opt.take().expect("cannot call `FnOnce` more than once");
                boxed.call_box();
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::Deferred;

    #[test]
    fn smoke_on_stack() {
        let a = [0u64; 1];
        let mut d = Deferred::new(move || drop(a));
        d.call();
    }

    #[test]
    fn smoke_on_heap() {
        let a = [0u64; 10];
        let mut d = Deferred::new(move || drop(a));
        d.call();
    }

    #[test]
    #[should_panic(expected = "cannot call `FnOnce` more than once")]
    fn twice_on_stack() {
        let a = [0u64; 1];
        let mut d = Deferred::new(move || drop(a));
        d.call();
        d.call();
    }

    #[test]
    #[should_panic(expected = "cannot call `FnOnce` more than once")]
    fn twice_on_heap() {
        let a = [0u64; 10];
        let mut d = Deferred::new(move || drop(a));
        d.call();
        d.call();
    }

    #[test]
    fn string() {
        let a = "hello".to_string();
        let mut d = Deferred::new(move || assert_eq!(a, "hello"));
        d.call();
    }

    #[test]
    fn boxed_slice_i32() {
        let a: Box<[i32]> = vec![2, 3, 5, 7].into_boxed_slice();
        let mut d = Deferred::new(move || assert_eq!(*a, [2, 3, 5, 7]));
        d.call();
    }
}
```
</details>

# Drawbacks

None.

# Alternatives

Keep `defer_free`.

# Unresolved questions

### Do we need `defer_drop`?

It turns out we can replace `defer_drop` with `defer`, too:

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

But this now is a bit unwieldy...

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

But even this is a bit sketchy, and we'd probably prefer conversion to `Ptr<'unsafe>` instead.
A RFC for `unsafe` lifetime was [proposed](https://github.com/rust-lang/rfcs/pull/1918)
and postponed.

Unless we figure out how to make destruction of `Ptr` using `defer` more ergonomic and safe
at the same time, this is left as an unresolved question.

# Summary

This is a proposal for a better atomics API in the context of epoch GC.

The API deals with pointers to heap-allocated objects protected by epoch GC.
There are two kinds of pointers:

1. Atomic pointers that usually live on the heap.
2. Pointers that live on the stack. These can get loaded from and stored into atomic pointers.

# Motivation

Crossbeam's current API suffers from the following problems:

1. It is unsound (safe code allows data races).
2. It doesn't provide complete flexibility (e.g. missing methods that operate with raw pointers).
3. It doesn't provide complete control over performance (e.g. CAS doesn't load on failure).
4. It lacks pointer tagging (necessary for implementing linked lists).
5. It could be more ergonomic.

This proposal aims to solve all problems. Solving all of them at once requires
careful balance of many tradeoffs. It's a difficult thing to do, but it will be
an important foundation for future work.

A more detailed analysis follows...

### Pointer types

Crossbeam currently has the following types for dealing with atomics:

1. `Atomic` - equivalent to `AtomicPtr`.
2. `Owned` - equivalent to `Box`.
3. `Shared` - equivalent to references.

There is also a type called `Guard`. The existence of a guard is proof that
the current thread is pinned. In other words, `epoch::pin()` returns a `Guard`
that can be passed to methods that require such proof.

There is an ergonomics issue that has to do with methods in `Atomic`
accepting and returning `Option<Owned<T>>` and `Option<Shared<T>>` types.

This is a subjective assertion, but taken from experience: those optional
types offer few benefits. They mostly feel clunky and unnecessary.

Consider the following methods in the current API:

```rust
fn cas(&self, old: Option<Shared<T>>, new: Option<Owned<T>>, ord: Ordering)
    -> Result<(), Option<Owned<T>>>;
fn cas_shared(&self, old: Option<Shared<T>>, new: Option<Shared<T>>, ord: Ordering)
    -> bool;
```

First of all, there are two ways of installing a null pointer:

```rust
atomic.cas(old, None, SeqCst);
atomic.cas_shared(old, None, SeqCst);
```

There is no use for `Option<Owned<T>>` in the first method. It can simply become:

```rust
fn cas(&self, old: Option<Shared<T>>, new: Owned<T>, ord: Ordering)
    -> Result<(), Owned<T>>;
```

Also, let's make `Shared<T>` encapsulate "optionalness" from within, so that it
can by itself represent null references:

```rust
fn cas(&self, old: Shared<T>, new: Owned<T>, ord: Ordering) -> Result<(), Owned<T>>;
fn cas_shared(&self, old: Shared<T>, new: Shared<T>, ord: Ordering) -> bool;
```

Now, to install a null pointer using CAS, we can do:

```rust
atomic.cas_shared(old, Shared::null(), SeqCst);
```

This simplification removes some redundancy. It also makes the API somewhat easier
to use by eliminating the need to wrap shared references into `Option`s (subjective opinion).

Moreover, handling "optionalness" from within is necessary to represent tagged
null pointers. Pointer tagging is described in the following section.

One drawback is that we've just lost a way to represent a non-nullable shared pointer.
But in practice this is not a problem. Introducing such a type is not difficult,
but quite possibly not worth the fuss at all (also subjective opinion).

Since `Shared` is now nullable, it can be created without a pin by calling `Shared::null()`.
In other words, the existence of a `Shared` used to be proof that the current thread
is pinned. But now it isn't anymore, so methods like `cas_and_ref` must take a guard:

```rust
fn cas_and_ref<'g>(&self, old: Shared<T>, new: Owned<T>, ord: Ordering, _: &'g Guard)
    -> Result<Shared<'g, T>, Owned<T>>
```

### Pointer tagging

There is an [open PR](https://github.com/crossbeam-rs/crossbeam/pull/70) that
introduces tagged atomic pointers. Most pointers have a few unused least
significant bits that can contain some information. This is useful for building
list-based data structures, e.g. linked lists and skiplists.

A question arises: do we need distinct types for normal atomics and tagged atomics?
As [coco](https://github.com/stjepang/coco) demonstrates, this distinction is not necessary.
We can just make tagging a built-in feature of `Atomic`/`Owned`/`Shared`, so
pointers that don't use tagging get tagged with zero.
Tagging is a feature that doesn't get in the way when it is not used.

One drawback of having tagging built-in is that tagged pointers have some overhead:

1. When creating a new owned pointer, we must assert that it is aligned.
2. When loading and dereferencing a pointer, we must unset the least significant bits.

But this overhead is really negligible (in both code size and runtime cost) and
boils down to simple bitwise operations.
We must also enforce proper alignment on pointers, but this is not a serious limitation.

If one doesn't need tagging, they don't even have to know about it. Tagging is a
very simple feature - it introduces just two methods on `Owned` and `Shared`:

```rust
fn tag(&self) -> usize;
fn with_tag(self, tag: usize) -> Self;
```

### Soundness hole

Aaron Turon's [blog post](https://aturon.github.io/blog/2015/08/27/epoch/)
that introduced Crossbeam states:

*"After we take a snapshot, we can dereference it without using unsafe,
because the guard guarantees its liveness."*

This is incorrect. We can safely dereference it only if we chose the right
memory orderings for loads and stores.

Consider the following scenario. There are two threads.
The first thread performs:

```rust
a.cas(Shared::null(), Owned::new(777), Relaxed, &guard);
```

The second thread performs:

```rust
println!("{}", a.load(Relaxed, &guard).as_ref().unwrap());
```

The number that is allocated on the heap (777) might not get printed because
the CAS in the first thread is not synchronized with the load in the second thread.
The memory orderings must be at least `Release` and `Acquire`, respectively.

By using relaxed orderings we can access unsynchronized non-atomic data within safe code.
This is unsound, as Rust must not allow data races (nor reading uninitialized data) in safe code.

It's possible to fix the problem by restricting the API for loads to only accept
`Acquire` or stronger, restricting stores to only accept `Release` or stronger,
and restricting CASes to only accept `AcqRel` or stronger. That
would guarantee freedom from data races. However, sometimes we do need more relaxed
orderings. Designing a flexible, ergonomic, and sound API that allows truly safe
`as_ref()` is too difficult.

Dereferencing is the crucial problem here: it is unsafe without proper
synchronization. For that reason, `as_ref()` will be an unsafe method.

This is perhaps disappointing news. One might ask: if Crossbeam doesn't even
permit safe dereferencing anymore, what does Rust as a language bring to the
table at all? Well, Rust disallows misuse of the epoch GC in multiple ways:

1. It enforces correct use of owned pointers. For example, CAS consumes it
on success and returns it back on failure. This is achieved through Rust's
move semantics and affine types.
2. It enforces correct use of shared pointers. Shared pointers are valid
only when the current thredad is pinned. This is achieved with the help of
Rust's lifetimes.

### Destructors

Destructors for concurrent data structures usually just iterate over all
nodes and deallocate them one by one. For example:

```rust
impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        let guard = epoch::pin();

        let mut curr = self.head.load(Relaxed, &guard);
        while !curr.is_null() {
            unsafe {
                let next = curr.as_ref().unwrap().next.load(Relaxed, &guard);
                drop(Box::from_raw(curr.as_raw()));
                curr = next;
            }
        }
    }
}
```

The problem with that piece of code is that it pins the current thread for
the entire duration of destruction. Destruction of big stacks might take
a long time, so this is pretty bad. It's possible to fix the problem by
destroying just a handful of nodes in one go and then re-pinning the thread.

But that trick might be more difficult to perform with other data structures,
especially tree-like ones. It is desirable to load atomics in destructors
without pinning the current thread.

It'd be nice to have a way to perform "fake" thread pinning. If we
know that the current thread is the only one accessing the data structure,
pinning is completely unnecessary.

# Detailed design

Now that we've discussed the problems in the current API, let's take a
step back and design a new one from scratch. Let's start with a blank page...

Epoch-based GC will have two crucial types:

1. `Atomic<T>` - an atomic pointer to a heap-allocated `T`. It usually lives on the heap.
2. `Ptr<'scope, T>` - a pointer to a heap-allocated `T`. It lives on the stack within `'scope`.

Loading an `Atomic<T>` returns a new `Ptr<'scope, T>`, which can be dereferenced
(if it isn't null). Example:

```rust
let a = Atomic::new(7); // Atomic pointer to a heap-allocated number 7.
let p = a.load(SeqCst);
```

But this doesn't work just yet! Epoch-based GC is all about scopes.
When working with heap-allocated objects, we don't want other threads to concurrently
destroy them under our feet. Let's say we'd like to print the number:

```rust
let p = a.load(SeqCst);
println!("{}", *p.as_ref().unwrap());
```

Between the `load` and `println` we really don't want some other thread to free the
heap-allocated number. There's a way to say to the GC: "please don't destroy any
garbage created during the life of this scope until it ends":

```rust
epoch::pin(|scope| {
    let p = a.load(Seqcst, scope);
    println!("{}", p.as_ref().unwrap());
})
```

Now, if another thread decides to free the heap-allocated number at a time between
the `load` and `println`, freeing will be simply deferred. The garbage collector
keeps track of such scopes and will destroy garbage later at a safe time.

Note that all `Ptr`s have a lifetime in their type: `Ptr<'scope, T>`, which
indicates the scope in which they're usable. Loading an atomic requires a scope,
which binds the returned `Ptr` to it.

The sample code still doesn't compile because `as_ref()` is unsafe. Dereferencing
is safe only if you use memory orderings to properly synchronize writes to and
reads from heap-allocated objects. Finally, this will compile:

```rust
epoch::pin(|scope| {
    let p = a.load(Seqcst, scope);
    println!("{}", unsafe { p.as_ref() }.unwrap());
});
```

If you're wondering why we're calling `unwrap()` here, that is because the pointer
might be null. Conversion to a reference succeeds only if it is not null.

Scopes created by `epoch::pin` must be short-lived, otherwise the GC might get
stalled for a long time and keep growing too much garbage.

Sometimes, however, we'd like to have longer-lived scopes in which we know our
thread is the only one accessing atomics. This is true e.g. when destructing a
big data structure, or when constructing it from a long iterator.
In such cases we don't need to be overprotective because there is no fear of
other threads concurrently destroying objects. Then we can do:

```rust
impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        unsafe {
            epoch::unprotected(|scope| {
                let mut curr = self.head.load(Relaxed, scope);
                while !curr.is_null() {
                    let next = curr.deref().next.load(Relaxed, scope);
                    drop(Box::from_raw(curr));
                    curr = next;
                }
            })
        }
    }
}
```

Function `epoch::unprotected` is unsafe because we must promise that no other thread
is accessing the `Atomic`s and objects at the same time. The function is safe to
use in the following cases only:

* Our thread is the only one accessing the atomics.
* No thread (including our own) is modifying the atomics.

Just like with the safe `epoch::pin` function, unprotected use of atomics is
enclosed within a scope so that pointers created within it don't leak out or get
mixed with pointers from other scopes.

How do we store new objects into `Atomic`s?
For that, there is type `Owned<T>`, which is almost identical to `Box<T>`.
In fact, `Owned<T>` can even be constructed directly from `Box<T>`.

We can convert an `Owned<T>` into `Ptr<'scope, T>`:

```rust
let mut owned = Owned::new(Node {
    value: value,
    next: Atomic::null(),
});

epoch::pin(|scope| {
    let node = owned.into_ptr(scope);
    // ...
})
```

Or, we can use compare-and-swap to install new objects into atomic pointers.
Here's how one might push a new `owned` node onto a stack:

```rust
epoch::pin(|scope| {
    let mut head = self.head.load(Acquire, scope);
    loop {
        owned.next.store(head, Relaxed);
        match self.head.compare_and_swap_weak_owned(head, owned, AcqRel, scope) {
            Ok(_) => break,
            Err((h, o)) => {
                head = h;
                owned = o;
            }
        }
    }
})
```

### The new interface

First, there is the type `Atomic`. Note that a scope has to be passed only
when the method returns a newly loaded pointer from the `Atomic`.

Methods `compare_and_swap_owned` and `compare_and_swap_weak_owned` in case of
failure return both the owned pointer that was not installed and the current
value stored in the atomic. This makes CAS loops more efficient by avoiding
the need for a fresh load.

```rust
struct Atomic<T> { ... }

impl<T> Atomic<T> {
    pub fn null() -> Self;
    pub fn new(t: T) -> Self;
    pub fn from_owned(owned: Owned<T>) -> Self;
    pub fn from_ptr(ptr: Ptr<T>) -> Self;

    pub fn load<'scope>(&self, ord: Ordering, _: &'scope Scope) -> Ptr<'scope, T>;

    pub fn store(&self, new: Ptr<T>, ord: Ordering);
    pub fn store_owned(&self, new: Owned<T>, ord: Ordering);

    pub fn swap<'scope>(&self, new: Ptr<T>, ord: Ordering, _: &'scope Scope)
        -> Ptr<'scope, T>;

    pub fn compare_and_swap<'scope>(
        &self,
        current: Ptr<T>,
        new: Ptr<T>,
        ord: Ordering,
        _: &'scope Scope,
    ) -> Result<(), Ptr<'scope, T>>;

    pub fn compare_and_swap_weak<'scope>(
        &self,
        current: Ptr<T>,
        new: Ptr<T>,
        ord: Ordering,
        _: &'scope Scope,
    ) -> Result<(), Ptr<'scope, T>>;

    pub fn compare_and_swap_owned<'scope>(
        &self,
        current: Ptr<T>,
        new: Owned<T>,
        ord: Ordering,
        _: &'scope Scope,
    ) -> Result<Ptr<'scope, T>, (Ptr<'scope, T>, Owned<T>)>;

    pub fn compare_and_swap_weak_owned<'scope>(
        &self,
        current: Ptr<T>,
        new: Owned<T>,
        ord: Ordering,
        _: &'scope Scope,
    ) -> Result<Ptr<'scope, T>, (Ptr<'scope, T>, Owned<T>)>;
}
```

Next up is `Owned`. It is basically just a taggable `Box<T>`.
An interesting method is `into_ptr`, which lifts an `Owned<T>` into
the "epoch universe".

```rust
struct Owned<T> { ... }

impl<T> Deref for Owned<T> { ... }
impl<T> DerefMut for Owned<T> { ... }

impl<T> Owned<T> {
    pub fn new(t: T) -> Self;
    pub fn from_box(b: Box<T>) -> Self;

    pub fn as_raw(&self) -> *mut T;
    pub fn into_ptr<'scope>(self, _: &'scope Scope) -> Ptr<'scope, T>;

    pub fn tag(&self) -> usize;
    pub fn with_tag(self, tag: usize) -> Self;
}
```

Finally, `Ptr<'scope, T>` acts like a taggable `Option<&'scope T>`. Most
interesting methods are probably `deref` and `as_ref`, which dereference
the pointer. As explained above, they are unsafe because we must promise
that load/store/CAS are properly synchronized.

```rust
struct Ptr<'scope, T: 'scope> { ... }

impl<'scope, T> Clone for Ptr<'scope, T> { ... }
impl<'scope, T> Copy for Ptr<'scope, T> { ... }

impl<'scope, T> Ptr<'scope, T> {
    pub fn null() -> Self;
    pub unsafe fn from_raw(raw: *mut T) -> Self;

    pub fn is_null(&self) -> bool;
    pub fn as_raw(&self) -> *mut T;
    pub unsafe fn deref(&self) -> &'scope T;
    pub unsafe fn as_ref(&self) -> Option<&'scope T>;

    pub fn tag(&self) -> usize;
    pub fn with_tag(&self, tag: usize) -> Self;
}
```

For comparison, here's the old interface for:

* [`Atomic`](https://docs.rs/crossbeam/0.2.10/crossbeam/mem/epoch/struct.Atomic.html)
* [`Owned`](https://docs.rs/crossbeam/0.2.10/crossbeam/mem/epoch/struct.Owned.html)
* [`Shared`](https://docs.rs/crossbeam/0.2.10/crossbeam/mem/epoch/struct.Shared.html)

### Example: linked list

Here's an example of linked list traversal.
Deleted nodes have their `next`-pointers tagged with number 1.

```rust
epoch::pin(|scope| {
    'retry: loop {
        let mut pred = self.head;
        let mut curr = pred.load(Acquire, scope);

        while let Some(c) = unsafe { curr.as_ref() } {
            let succ = c.next.load(Acquire, scope);

            // Is the current node marked as deleted?
            if succ.tag() == 1 {
                let succ = succ.with_tag(0);

                // Try unlinking it.
                if pred.compare_and_swap(curr, succ, AcqRel, scope).is_err() {
                    // Failed. Traverse again from the beginning.
                    continue 'retry;
                }

                curr = succ;
            } else {
                // We just traversed `c`!

                // Move to the next node.
                pred = &c.next;
                curr = succ;
            }
        }

        // Done! Successfully traversed the whole list.
        break;
    }
})
```

### Scopes or guards?

Why does `epoch::pin` take a closure rather than returning a `Scope`,
or a `Guard` as before?

Well, let's get this out of the way: guards as they are currently
implemented in Crossbeam are unsound. Consider the following program:

```rust
struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        GUARD.with(|guard| {
            let guard = guard.borrow();
            let guard = guard.as_ref().unwrap();

            // Unprotected load!!!
            println!("{}", *ATOMIC.load(SeqCst, guard).unwrap());
        });

        // Remove the guard and forget it.
        GUARD.with(|guard| mem::forget(guard.borrow_mut().take()));
    }
}

lazy_static! {
    static ref ATOMIC: Atomic<i32> = Atomic::new(7);
}

thread_local! {
    static GUARD: RefCell<Option<Guard>> = RefCell::new(None);
    static FOO: Foo = Foo;
}

fn main() {
    thread::spawn(|| {
        GUARD.with(|_| ());
        FOO.with(|_| ());
        GUARD.with(|guard| *guard.borrow_mut() = Some(epoch::pin()));

        // 1. `LOCAL_EPOCH` gets dropped. Thread is unregistered.
        // 2. `FOO` gets dropped. The number is loaded and printed on the screen.
        // 3. `GUARD` gets dropped. Nothing happens because we removed the guard when dropping `FOO`.
    }).join().unwrap();
}
```

Let's also add this to `LocalEpoch::drop` in the `mem::epoch::local` module:

```rust
println!("Unregister!");
```

Running the program outputs:

```
Unregister!
7
```

As we can see, the thread was first unregistered. Then we used the
thread-local guard to load a global atomic and print it on the screen.
While the thread was unregistered in the epoch GC we successfully
accessed an atomic, which is wrong and unsafe!

Guards are tricky to get completely right. They are surely more
ergonomic than closures, but closures more explicitly demonstrate
that something important is going on in a scope.
Closures are also more difficult to misuse.

Moreover, it's hard to explain why, but closures have shown slightly
better numbers in my benchmarks (anecdotal evidence).

Perhaps the decision is not clear-cut, but for the aforementioned
reasons closures are preferred over guards. They are more explicit,
harder to misuse, and easier to implement correctly.

# Drawbacks

We'll completely break compatibility with older versions of Crossbeam.

# Alternatives

1. Create a type distinct from `Atomic` for tagged atomics.

2. Make a distinction between nullable and non-nullable `Ptr`.

3. Some method names are very long, e.g. `compare_and_swap_weak_owned`. They
could be shortened to something like `cas_weak_owned`, but that'd reduce
some clarity. After all, C++ has even longer functions like
`atomic_compare_exchange_strong_explicit`.

# Unresolved questions

What if one needs an `Owned<T>` that uses custom functions for
allocation and deallocation? Should we provide a nice interface for
such cases?

Also, how do dynamically sized types interact with all this?

### Fat pointers and DSTs

Consider a node in a skiplist. It consists of: key, value, tower. The tower is
an array of atomic pointers. To save a level of indirection, it is wise to lay out
the entire tower inside the node. This means that nodes are dynamically sized.

Another example might be arrays backing hash-tables or Chase-Lev deques. They too
are dynamically sized so it might make sense to lay out the length together with
array's elements. B-tree nodes may likewise be dynamically sized.

Rust usually represents instances of DST types as fat pointers. This is problematic
in lock-free programming because fat pointers consist of two words. Atomic operations on
such types are possible on modern architectures (they have instructions for DWCAS),
but that is still not portable enough nor available on stable Rust.
There are also other reasons why one might want thin pointers with length stored
within the object, like performance (cache locality) and memory consumption.

For that reason `Atomic` will not attempt to support fat pointers.
Dynamically sized types are an interesting topic and may be discussed in more
detail in a future RFC.

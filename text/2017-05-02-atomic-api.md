# Summary

This is a proposal for a better atomics API in the context of epoch GC.

The API deals with pointers to heap-allocated objects protected by epoch GC.
There are two kinds of pointers:

1. Atomic pointers that live on the heap.
2. Pointer that live on the stack. These can get loaded from and stored into atomic pointers.

# Motivation

Crossbeam's current API has the following problems:

1. The API is unsound (safe code allows data races).
2. It lacks several methods to allow full flexibility (e.g. unsafe methods for raw pointer control).
3. It lacks pointer tagging (necessary for implementing linked lists).
4. It could be more ergonomic.

The API will be enriched, simplified, and the soundness holes will be fixed.

# Detailed design

Just an upfront emphasis: designing a good API is really difficult and comes
with many tradeoffs.

The problem is that it's too tempting to create a large API soup with a lot
of similar, but slightly different types and methods. This can easily become
a big confusing mess.

Please take that into consideration. It might be worth sacrificing some seemingly
useful methods or types in order to attain a good-looking API in the end.

The following sections contain a lot of hand-wavy language to illustrate
problems and discuss potential solutions. In the end a final detailed design
is proposed.

### Pointer types

Crossbeam currently has the following types for dealing with atomics:

1. `Atomic` (similar to `AtomicPtr`)
2. `Owned` (equivalent to `Box`)
3. `Shared` (equivalent to references)

There is also a type called `Guard`. In this text it is interchangeable with
`Pin` and is simply proof that the current thread is pinned. In other words,
pinning a thread gives a `Guard`/`Pin` that can be passed to methods that
require such proof. The distinction between these two types will be explained
in a future RFC, but for now they can be considered to be the same thing.

An ergonomics issue that comes up has to do with methods in `Atomic`
accepting and returning `Option<Owned<T>>` and `Option<Shared<T>>` types.

This is a subjective assertion, but taken from experience: those optional
types offer little benefit. Mostly they just feel clunky and unnecessary.

Consider the following methods in the current API:

```rust
fn cas(&self, old: Option<Shared<T>>, new: Option<Owned<T>>, ord: Ordering)
    -> Result<(), Option<Owned<T>>>;
fn cas_shared(&self, old: Option<Shared<T>>, new: Option<Shared<T>>, ord: Ordering)
    -> bool;
```

First of all, there are two ways to install a null pointer:

```rust
atomic.cas(old, None, SeqCst);
atomic.cas_shared(old, None, SeqCst);
```

There is no use for `Option<Owned<T>>` in the first method. It can become simply:

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

Now, install a null pointer using CAS, we can do:

```rust
atomic.cas_shared(old, Shared::null(), SeqCst);
```

This simplification removes some redundancy. It also makes the API somewhat easier
to use by eliminating the need to wrap shared references into `Option`s (subjective opinion).

Moreover, handling "optionalness" from within is necessary to represent tagged
null pointers. Pointer tagging is described in the following section.

One drawback is that we've just lost a way to represent a non-nullable shared pointer.
But in practice this is not a problem. Introducing such a type is not difficult,
but quite possibly not worth the fuss at all (subjective opinion).

Since `Shared` is now nullable, it can be created without a pin by calling `Shared::null()`.
In other words, the existence of a `Shared` used to be proof that the current thread
is pinned. But now it isn't anymore, so methods like `cas_and_ref` must take a pin/guard:

```rust
fn cas_and_ref<'p>(&self, old: Shared<T>, new: Owned<T>, ord: Ordering, _: &'p Pin)
    -> Result<Shared<'p, T>, Owned<T>>
```

### Pointer tagging

There is an [open PR](https://github.com/crossbeam-rs/crossbeam/pull/70) that
introduces tagged atomic pointers. Most pointers have a few unused least
significant bits that can contain some information. This is necessary for building
list-based data structures, e.g. linked lists and skiplists.

A question arises: do we need distinct types for normal atomics and tagged atomics?
As [coco](https://github.com/stjepang/coco) demonstrates, this distinction is not necessary.
We can just make tagging a built-in feature of `Atomic`/`Owned`/`Shared`, so
pointers that don't use tagging get tagged with zero.
Tagging is a feature that doesn't get in the way when it is not used.

A drawback might be that tagged pointers have some overhead:

1. When creating a new owned pointer, we must assert that it is aligned.
2. When loading and dereferencing a pointer, we must unset the least significant bits.

But this overhead is really negligible and boils down to simple bitwise operations.
We also enforce proper alignment on pointers, but this is not a serious limitation.

If one doesn't need tagging, they don't even have to know about it. Tagging is a
very simple feature - it introduces just two methods on `Owned` and `Shared`:

```rust
fn tag(&self) -> usize;
fn with_tag(self, tag: usize) -> Self;
```

### Soundness hole

Aaron Turon's [blog post](https://aturon.github.io/blog/2015/08/27/epoch/)
that introduced Crossbeam states:

"After we take a snapshot, we can dereference it without using unsafe,
because the guard guarantees its liveness."

This is incorrect. We can safely dereference only if we chose the right
memory orderings for loads and stores.

Consider the following scenario. There are two threads.
The first thread performs:

```rust
a.cas(Shared::null(), Owned::new(777), Relaxed, pin);
```

The second thread performs:

```rust
println!("{}", a.load(Relaxed, pin).as_ref().unwrap());
```

The number that is allocated on the heap (777) might not get printed because
the CAS in the first thread is not synchronized with the load in the second thread.
The memory orderings must be at least `Acquire` and `Release`, respectively.

By using relaxed orderings we can access unsynchronized non-atomic data within safe code.
This is unsound, as Rust must not allow data races (or reading uninitialized data) in safe code.

It's possible to fix the problem by restricting the API for loads to only accept
`Acquire` or stronger, and stores to only accept `Release` or stronger. That
would guarantee freedom from data races. However, sometimes we do need relaxed
orderings, and designing a sound API that allows safe `as_ref()` is difficult.

Dereferencing is the crucial problem here: it is unsafe without proper
synchronization. For that reason, `as_ref()` will be an unsafe method.

This is perhaps disappointing news. One might ask: if Crossbeam doesn't even
permit safe dereferencing, what does Rust bring to the table at all?
Well, Rust disallows misuse of the epoch GC in multiple ways:

1. It enforces correct use of owned pointers. For example, CAS consumes it
on success and returns it back on failure. This is achieved through Rust's
move semantics and affine types.
2. It enforces correct use of shared pointers. Shared pointers are valid
only when the current thredad is pinned. This is achieved with the help of
Rust's lifetimes.

### Destructors

Destructors for concurrent data structures usually just iterate over all
nodes and deallocate them. For example:

```rust
impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        let pin = unsafe { &epoch::assert_ownership() };

        let mut curr = self.head.load(Relaxed, pin);
        while !curr.is_null() {
            let next = curr.deref().next.load(Relaxed, pin);
            drop(Box::from_raw(curr.as_raw()));
            curr = next;
        }
    }
}
```

The problem with that piece of code is that it pins the current thread for
the entire duration of destruction. Destruction might take a long time, so
this is pretty bad. It's possible to fix the problem by
destroying just a handful of nodes and then pinning the current thread again.

That trick might be difficult to perform with some other data structures,
especially tree-like ones. Since destructors are wildly unsafe anyway, it
is desirable to load atomics in them without pinning the current thread.

For that reason, there will also be method `load_raw` that returns the
raw pointer (`*mut T`) and the tag (`usize`). Now the destructor can look
like this:

```rust
impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        let mut curr = self.head.load_raw(Relaxed).0;
        while !curr.is_null() {
            unsafe {
                let next = (*curr).next.load_raw(Relaxed).0;
                drop(Box::from_raw(curr));
                curr = next;
            }
        }
    }
}
```

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
but that is still not portable enough. There are also other reasons why one might
want thin pointers with length stored within the object, like performance (cache
locality) and memory consumption.

For that reason `Atomic` will not attempt to support fat pointers.
Dynamically sized types are an interesting topic and will be discussed in more
detail in a future RFC.

### The proposed API

For comparison, here's old current interface for:

* [`Atomic`](https://docs.rs/crossbeam/0.2.10/crossbeam/mem/epoch/struct.Atomic.html)
* [`Owned`](https://docs.rs/crossbeam/0.2.10/crossbeam/mem/epoch/struct.Owned.html)
* [`Shared`](https://docs.rs/crossbeam/0.2.10/crossbeam/mem/epoch/struct.Shared.html)

The new interface follows.

First, there is the type `Atomic`. Note that a pin/guard has to be passed only
when the method returns a newly loaded pointer from the `Atomic`.

Methods `cas_owned` and `cas_weak_owned` in case of failure return both the owned
pointer that was not installed and the current value stored in the atomic. This makes
CAS loops more efficient by avoiding the need for a fresh load.

```rust
struct Atomic<T> { ... }

impl<T> Atomic<T> {
    pub fn null() -> Self;
    pub fn from_owned(owned: Owned<T>) -> Self;
    pub fn from_shared(shared: Shared<T>) -> Self;

    pub fn load<'p>(&self, ord: Ordering, _: &'p Pin) -> Shared<'p, T>;
    pub fn load_raw(&self, ord: Ordering) -> (*mut T, usize);

    pub fn store(&self, new: Shared<T>, ord: Ordering);
    pub fn swap<'p>(&self, new: Shared<T>, ord: Ordering, _: &'p Pin) -> Shared<'p, T>;

    pub fn cas<'p>(
        &self,
        current: Shared<T>,
        new: Shared<T>,
        ord: Ordering,
        _: &'p Pin,
    ) -> Result<(), Shared<'p, T>>;

    pub fn cas_weak<'p>(
        &self,
        current: Shared<T>,
        new: Shared<T>,
        ord: Ordering,
        _: &'p Pin,
    ) -> Result<(), Shared<'p, T>>;

    pub fn cas_owned<'p>(
        &self,
        current: Shared<T>,
        new: Owned<T>,
        ord: Ordering,
        _: &'p Pin,
    ) -> Result<Shared<'p, T>, (Shared<'p, T>, Owned<T>)>;

    pub fn cas_weak_owned<'p>(
        &self,
        current: Shared<T>,
        new: Owned<T>,
        ord: Ordering,
        _: &'p Pin,
    ) -> Result<Shared<'p, T>, (Shared<'p, T>, Owned<T>)>;
}
```

Next up is `Owned`. It is basically just a taggable `Box<T>`.
An interesting method is `into_shared`, which lifts an `Owned<T>` into
the "epoch universe".

```rust
struct Owned<T> { ... }

impl<T> Deref for Owned<T> { ... }
impl<T> DerefMut for Owned<T> { ... }

impl<T> Owned<T> {
    pub fn new(t: T) -> Self;
    pub fn from_box(b: Box<T>) -> Self;

    pub fn as_raw(&self) -> *mut T;
    pub fn into_shared<'p>(self, _: &'p Pin) -> Shared<'p, T>;

    pub fn tag(&self) -> usize;
    pub fn with_tag(self, tag: usize) -> Self;
}
```

Finally, `Shared<'p, T>` acts like a taggable `&'p T`. Most interesting
methods are probably `deref` and `as_ref`, which dereference the pointer.
As explained above, they are unsafe.

```rust
struct Shared<'p, T: 'p> { ... }

impl<'p, T> Clone for Shared<'p, T> { ... }
impl<'p, T> Copy for Shared<'p, T> { ... }

impl<'p, T> Shared<'p, T> {
    pub fn null() -> Self;
    pub unsafe fn from_raw(raw: *mut T) -> Self;

    pub fn is_null(&self) -> bool;
    pub fn as_raw(&self) -> *mut T;
    pub unsafe fn deref(&self) -> &'p T;
    pub unsafe fn as_ref(&self) -> Option<&'p T>;

    pub fn tag(&self) -> usize;
    pub fn with_tag(&self, tag: usize) -> Self;
}
```

### Examples

Here are some interesting examples of the API in use.

Storing an `Owned` into `Atomic`:

```rust
atomic.store(Owned::new(t).into_shared(pin), SeqCst);
```

Pushing an element onto the Treiber stack:

```rust
fn push(&self, value: T) {
    let mut node = Owned::new(Node {
        value: value,
        next: Atomic::null(),
    });

    epoch::pin(|pin| { // Or: let pin = epoch::pin();
        let mut head = self.head.load(Acquire, pin);
        loop {
            node.next.store(head, Relaxed);
            match self.head.cas_weak_owned(head, node, AcqRel, pin) {
                Ok(_) => break,
                Err((h, n)) => {
                    head = h;
                    node = n;
                }
            }
        }
    })
}
```

Here's an example of linked list traversal.
Deleted nodes have their `next`-pointers tagged with number 1.

```rust
'retry: loop {
    let mut pred = head;
    let mut curr = pred.load(Acquire, pin);

    while let Some(c) = unsafe { curr.as_ref() } {
        let succ = c.next.load(Acquire, pin);

        // Is the current node marked as deleted?
        if succ.tag() == 1 {
            let succ = succ.with_tag(0);
            // Try unlinking it.
            if pred.cas(curr, succ, AcqRel, pin).is_err() {
                // Failed. Traverse again from the beginning.
                continue 'retry;
            }
            curr = succ;
        } else {
            // Move to the next node.
            pred = &c.next;
            curr = succ;
        }
    }
    break;
}
```

# Drawbacks

We'll break compatibility with old versions of Crossbeam.

# Alternatives

* Create a type distinct from `Atomic` for tagged atomics.
* Make a distinction between nullable and non-nullable `Shared`.

# Unresolved questions

How do dynamically sized types actually interact with all this?
In some cases they are really necessary.

What if one needs an `Owned<T>` that uses custom functions for
allocation and deallocation? Should we provide a nice interface for
such cases?

# Summary

This RFC introduces the `Collector` API for supporting diversified use cases.


# Motivation

As the Crossbeam project is growing, users want more diversified API than the existing one.

First, majority of users will be happy with the current API, which interacts with the default
singleton garbage collector.

Second, some users may want to create their own garbage collector, and use the collector via a
*handle* to it. Since the current API provides only one garbage collector, Crossbeam currently does
not support this use case.

Lastly, some users want to embed a garbage collector in another systems library, e.g. memory
allocator and thread management environment.  Note that it is often a part of the standard library,
thus this use case requires (at least a part of) Crossbeam to be implemented in the `#[no_std]`
environment. Likewise, Crossbeam currently does not support this use case.

In order to support these diversified use cases, this author proposes to introduce the **`Collector`
API**.



# Detailed design

FYI, this RFC is fully implemented in [this
branch](https://github.com/jeehoonkang/crossbeam-epoch/tree/handle). You can find a discussion on
this branch in [this PR](https://github.com/crossbeam-rs/crossbeam-epoch/pull/21).



## `Collector` API

In this layer of API, the public struct `Collector` is a general-purpose garbage collector with its
own global epoch, and the public struct `Handle` is an abstraction of a participant in a collector.
`Collector` is a counted reference to a the global data for garbage collection (`Global`), and
`Handle` is a tuple of a counted reference to a global data and a local data (`Local`):

```rust
pub struct Collector(Arc<Global>);

pub struct Handle {
    global: Arc<Global>,
    local: Local,
}
```

You can create a new collector by creating a new `Global` data, and add a handle to a collector by
sharing its `Global` data.  Using a `Handle`, you can create a `Scope` and pass it to a given
function:

```rust
impl Collector {
    pub fn new() -> Self;
    pub fn handle(&self) -> Handle;
}

impl Handle {
    pub fn pin<F, R>(&self, f: F) 
    where F: FnOnce(&Scope) -> R {
        self.local.pin(&self.global, f)
    }

    ... // other methods
}
```

Note that `Handle::pin()`, on contrary to `Local::pin()`, does not require a reference to a
`Global`, because `Handle` already has a reference.

See [the
implementation](https://github.com/jeehoonkang/crossbeam-epoch/blob/handle/src/collector.rs) for
more details.


## The default garbage collector API

There will be the default collector and per-thread handle to the default collector:

```rust
lazy_static! { pub COLLECTOR: Collector = Collector::new(); }
thread_local! { pub HANDLE: Handle = COLLECTOR.handle(); }

pub fn pin<F, R>(f: F) 
where F: FnOnce(&Scope) -> R {
    HANDLE.with(|handle| { handle.pin(f) })
}
```

See [the implementation](https://github.com/jeehoonkang/crossbeam-epoch/blob/handle/src/default.rs)
for more details.



# Alternatives

`&'scope Scope` vs. `Scope<'scope>`: Currently we pass `&'scope Scope` as a witness that the current
participant is pinned. The reference type may incur runtime overhead of nested indirection, but this
author believes the overhead if bearable. You may avoid this overhead by using `Scope<'scope>`
instead as the witness, but it is an unorthodox choice.

`Arc<Global>` vs. `&Global`: The proposed `Collector` is implemented as `Arc<Global>`, which incurs
a runtime overhead of reference counting.  This author believes the overhead is negligible, because
handle creation is likely in the cold path.  Using `&Global` instead of `Arc<Global>` might
eliminate the runtime cost, but it will significantly complicate the API.



# Unresolved questions

Both the `Collector` API and the default collector relies on the hidden `Global` and `Local`
structs. It might be beneficial to expose this internal API for the last bit of optimization, even
though this API is significantly more complicated than the `Collector` API. Unfortunately, currently
we are short of concrete use cases for precisely evaluation of the tradeoff.

For `#[no_std]` environment, a long-term plan would be separating out what is dependent on `std`,
namely the default collector API, and what is not, namely the `Collector` API. The proposed
`Collector` API is almost `std`-free except for the fact that (1) the `Collector` API uses
`std::sync::Arc`, and (2) they rely on the data structures in the standard library. This author
believes we can easily make it fully independent from `std` by (1) re-implementing `Arc` in
`#[no_std]`, and (2) using a `#[no_std]` data structures after [the allocator trait is
stabilized](https://github.com/rust-lang/rust/issues/32838).

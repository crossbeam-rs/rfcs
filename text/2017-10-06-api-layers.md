# Summary

This RFC introduces three layers of API for supporting diversified use cases.


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

In order to support these diversified use cases, this author proposes to generalize the API by
introducing an API layer for each use case: (1) **the default garbage collector API** for the first
use case; (2) **`Collector` API** for the second one; and (3) **internal API** for the third one.



# Detailed design

FYI, this RFC is fully implemented in [this
branch](https://github.com/jeehoonkang/crossbeam-epoch/tree/handle). You can find a discussion on
this branch in [this PR](https://github.com/crossbeam-rs/crossbeam-epoch/pull/21).


## Internal API

This is the lowest-level API to Crossbeam. First, we define a public struct `Global` and
`Local` that contain the global and local data for garbage collection:

```rust
struct Global {
  ...
}

struct Local {
  ...
}
```

And implement the following methods:

```rust
impl Global {
    pub fn new() -> Self;
}

impl Local {
    fn new(global: &Global) -> Self;

    pub fn pin<F, R>(&self, global: &Global, f: F)
    where F: for<'scope> FnOnce(Scope<'scope>) -> R;
    // fun f with a `Scope`

    pub unsafe fn finalize(&self, global: &Global);
    // deregisters itself from the global.

    ... // other methods
}
```

Note that `Local::pin()` requires a reference to a `Global`.

When you want to embed a garbage collector in another systems library, e.g. logger, you first define
its own `Global` and `Local` structs, and embed `epoch::Global` and `epoch::Local` in corresponding
structs, as follows:

```rust
pub mod Logger {

pub struct Global {
    epoch: epoch::Global,
    ... // more global data
}

pub struct Local {
    epoch: epoch::Local,
    ... // more local data
}

...

}
```

You can easily finish the internal logger API by implementing the methods of `Global` and
`Local` using `epoch`. You can build the `Logger` API and the default logger API in the same
way as below.

For a full example, see [this repository](https://github.com/jeehoonkang/handle-example-rs).


## `Collector` API

The public struct `Collector` is a general-purpose garbage collector with its own global epoch, and
the public struct `Handle` is an abstraction of a participant in a collector.  `Collector` is a
counted reference to a `Global`, and `Handle` is a tuple of a counted reference to a `Global` and a
`Local`:

```rust
struct Collector(Arc<Global>);

struct Handle {
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
    where F: for<'scope> FnOnce(Scope<'scope>) -> R {
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
lazy_static! { pub GLOBAL: Global = Global::new(); }
thread_local! { pub HANDLE: Handle = Handle::new(); }

struct Handle(Local);

impl Handle {
    fn new() -> Self {
        Self { 0: Local::new(&GLOBAL) }
    }
}

impl Deref for Handle {
    type Target = Local;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl Drop for Handle {
    fn drop(&mut self) {
        self.finalize(&GLOBAL);
    }
}

pub fn pin<F, R>(f: F) 
where F: for<'scope> FnOnce(Scope<'scope>) -> R {
    HANDLE.with(|handle| { handle.pin(&GLOBAL, f) })
}
```

See [the implementation](https://github.com/jeehoonkang/crossbeam-epoch/blob/handle/src/default.rs)
for more details.



# Alternatives

`Scope<'scope>` vs. `&'scope Scope`: Currently the type of scope is `Scope<'scope>`. It is possible
to use more traditional reference type `&'scope Scope`, but it may incur some runtime cost of double
indirection.

`Arc<Global>` vs. `&Global`: Currently `Collector` is implemented as `Arc<Global>`, which incurs a
runtime overhead of reference counting.  This author believes the overhead is negligible, because
handle creation is likely in the cold path.  Using `&Global` instead of `Arc<Global>` might
eliminate the runtime cost, but it will significantly complicates the API.



# Unresolved questions

for `#[no_std]` environment, a long-term plan would be separating out what is dependent on `std`,
namely the default collector API and the `Collector` API, and what is not, namely the internal
API. The proposed internal API is almost `std`-free except for the fact that `Global` and `Local`
relies on the data structures in the standard library. This author believes we can finish this job
once [the allocator trait is stabilized](https://github.com/rust-lang/rust/issues/32838).

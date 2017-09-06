# Summary

This is a proposal for generalizing the `Scope` struct. It primarily aims to optimize the
`unprotected` function. As a side benefit, we can easily support multiple instances of EBR-managed
region of shared memory.


# Motivation

The `unprotected` function provides a `Scope` without pinning the thread, where the caller thread is
either (1) the only one accessing atomics, or (2) no threads are modifying atomics (think:
reader-writer locks). Assuming this, we can **immediately** deallocate and drop objects in an
unprotected scope. However, even in an unprotected scope, the `defer_free` and `defer_drop`
functions defers freeing and dropping the objects until the scope is over. Even worse, if a lot of
pieces of garbage are created, they are moved to the global garbage queue.

Another motivation is the fact that currently we are supporting only a single instance of
EBR-managed region of shared memory. For example, a bad-behaving mutator can pin itself
indefinitely so that the global epoch cannot be advanced too long. In order to mitigate this
problem, we would like to support multiple instances of EBR so that a mutator in an instance cannot
bother the advancement of another instance's epoch.


# Detailed Design

This proposal is fully implemented
in [this branch](https://github.com/jeehoonkang/crossbeam-epoch/tree/unprotected).


## Generalizing `Scope`

In order to enhance the `unprotected` function, we propose:

- Creating
  the
  [`Scope` trait](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/realm.rs#L15),
  which provides the methods `Scope` currently has;

- Parameterizing `&'scope Scope` in lots of methods in `atomic.rs`
  with
  [`S: Scope<'scope>`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/atomic.rs#L209);

- Renaming the existing `Scope` struct
  into
  [`EpochScope`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/mutator.rs#L58),
  and
  [implementing `Scope` for `&EpochScope`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/mutator.rs#L276);
  and

- Making an empty
  struct
  [`UnprotectedScope`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/mutator.rs#L78),
  [implementing `Scope` for `UnprotectedScope`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/mutator.rs#L319),
  and
  [letting `unprotected` use `UnprotectedScope`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/mutator.rs#L200).

In the implementation of `Scope` for `UnprotectedScope`, the `defer*` functions immediately disposes
the garbage, and the `flush` function does nothing.


## Supporting Multiple Realms

In order to support multiple instances of EBR, we propose:

- Creating
  [`Realm` trait](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/realm.rs#L54),
  which represents an instance of EBR and provides the methods to access the mutator registries, the
  global queue, and the global epoch;

- Add
  a
  [`Realm` field](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/mutator.rs#L46) to
  the `Mutator` struct, which represents the realm it belongs to;

- Making an empty
  struct
  [`DefaultRealm`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/default.rs#L27),
  which implements `Realm` with static objects; and

- Making
  the
  [`UserRealm`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/realm.rs#L107)
  struct,
  which
  [implements `Realm`](https://github.com/jeehoonkang/crossbeam-epoch/blob/unprotected/src/realm.rs#L131) with
  its data.
  
You can use a custom `UserRealm` as follows:

```rust
use crossbeam_epoch::{Scope, UserRealm, Mutator};

let realm = UserRealm::new();
let mutator = Mutator::new(&realm);
mutator.pin(|scope| {
    unsafe {
        scope.defer(|| {
            println!("hello, world!");
        });
    }
});
```


# Drawbacks

Its interface is more complicated than before. However, I believe the implementation is clearer than
before. For example, now `Mutator` does not directly depend on the static data defined in
`default.rs` (renaming of
`global.rs`)](https://github.com/crossbeam-rs/crossbeam-epoch/pull/4#discussion_r130319009).
Furthermore, after the [impl Trait](https://github.com/rust-lang/rust/issues/34511) feature is
landed in the Rust compiler, we will no longer need to expose the implementations of `Scope`,
resulting in an interface as simple as that of today.



# Alternatives

Maybe the current interface is good enough.



# Unresolved questions

Is it really useful to use multiple instances of EBR? Is there any practical use cases of multiple
EBR instances?

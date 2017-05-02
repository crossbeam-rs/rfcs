# Summary

This is a proposal for a better atomics API in the context of epoch GC.

This API deals with atomic pointers to objects protected by epoch GC,
and also stack references to such objects.

# Motivation

Crossbeam currently has three types for dealing with pointers backed
by epoch GC: `Atomic`, `Owned`, `Shared`. The API is unsound, lacks
several methods to allow full flexibility, lacks pointer tagging, and
feel unergonomic when building data structures.

The API will be enriched, simplified, and the soundness holes will be
fixed.

# Detailed design

Just an upfront warning: designing a good API is really difficult.

The problem is that it's too easy to create a large API soup with a lot
of similar, but slightly different types and methods. This can easily
become a big confusing mess for their users.

Please take that into consideration. It might be worth sacrificing a few niche
use cases in order to achieve reasonable API simplicity.

### Pointer types

Crossbeam currently has types `Atomic`, `Owned`, and `Shared`. It turns out
that `Owned` is basically identical to `Box` and doesn't offer compelling
reasons for its existence. Likewise, `Shared` is just a wrapper around a
typical reference in Rust.

Another issue is that many methods in `Atomic` accept and return `Option<Owned<T>>`
and `Option<Shared<T>>` types. This is a subjective assertion, but taken from
experience: all those optional types offer little comfort. They mostly just 
feel clunky and unnecessary. The [coco][coco] crate instead has `Ptr<T>` that
acts just like `Option<Shared<T>>`. By encapsulating optional-ness from within,
it greatly simplifies dealing with atomics and makes the API more ergonomic.

It can be [tempting to also introduce](https://github.com/stjepang/coco/pull/1)
a non-nullable type like `Shared`, but
compelling reasons for its existence are yet to be seen (also a subjective opinion).

### Tagged pointers

There is an [open PR](https://github.com/crossbeam-rs/crossbeam/pull/70) that
introduces tagged atomic pointers. Most pointers have a few unused least
significant bits that can contain some information. This is necessary for building
list-based data structures, e.g. linked lists and skiplists.

As [coco][coco] shows, it's not necessary to have both `Atomic` and `TaggedAtomic`.
Instead of `Atomic` types, one can simply use `TaggedAtomic` and always set tags to
zero. By unifying these two types the API surface is greatly simplified (essentially
halved).

### Soundness hole

Consider the following scenario. There are two threads.

The first thread performs:

```rust
a.store(Some(Owned::new(777)), Relaxed);
```

The second thread performs:
```rust
println!("{}", a.load(Relaxed, guard).unwrap());
```

The number that is allocated on the heap (777) might not get printed because
the store in the first thread is not synchronized with the load in the second thread.
The memory orderings must be at least `Acquire` and `Release`, respectively.

By using relaxed orderings we can create a data race within safe code. This is
unsound, as Rust must not allow data races in safe code.

### Methods on `Atomic` type

For reference, [here](https://docs.rs/coco/0.1.0/coco/epoch/struct.Atomic.html)
is the current `Atomic` API in [coco][coco].

Besides simple constructors like `new`, `null`, `from_raw`, and so on, there is
a long list of methods that need to operate on `Atomic` type. There are
three classes of methods: those working with `Ptr` type (or `Shared`), with `Box`
(or `Owned`), and with raw pointers (`*mut T`).

* `load`
* `load_raw`
* `store`
* `store_box`
* `store_raw`
* `swap`
* `swap_box`
* `swap_raw`
* `cas`
* `cas_box`
* `cas_raw`
* `cas_weak`
* `cas_box_weak`
* `cas_raw_weak`

The question arises: which of these are safe, and which are unsafe? First, all methods
that work with raw pointers are unsafe. Others are safe depending on which memory
ordering is specified. For example, `load` is safe if it uses `Acquire` or `SeqCst`,
but unsafe if it uses `Relaxed`.

There are two obvious ways of forbidding `load` with `Relaxed` in safe code:

1. If `Relaxed` is specified, simply use `Acquire` instead.
2. Instead of `load` method expose two that don't take ordering argument at all:
`load_acquire` and `load_seq_cst`.

The first option silently has surprising behavior, and the second creates an API
explosion. By applying the second method to all listed methods except `*_raw*`, there
would now be 23 of them!

TODO: Help is needed to make a decision.

# Drawbacks

TODO: Why should we not do this?

# Alternatives

TODO: What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

TODO: What parts of the design are still TBD?

[coco]: https://github.com/stjepang/coco

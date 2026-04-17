+++
title = "What do Send and Sync actually mean?"
date = 2026-04-18
[taxonomies]
categories = ["Rust"]
+++
It's easy to meet the `Send + Sync` bounds when working with various (async) Rust codebases.
But what do they actually mean, and how are they implemented?
When is a type `Send` or `Sync`, and when is it not?

### Send
A type is `Send` if it is safe to send, or transfer ownership to another thread.
```rust
pub unsafe auto trait Send {}
```
Note that a type being `Send` doesn't provide any special implementation through the trait, but it's simply telling that the type satisfies a certain condition.
The `auto` keyword lets the compiler derive that a certain type is `Send` if all inner components are `Send`.

Thus, it's quite rare (and unsafe) to manually implement `Send` (or `!Send`) since you're basically insisting that a type is `Send` even though it contains non-`Send` types. (or that a type is not `Send` even though all the inner types are.)

### Sync
A type `T` is `Sync` if and only if `&T` is `Send`. In other words, a type is `Sync` if it is safe to pass shared references across threads. It's an auto marker trait just like `Send`.
```rust
pub unsafe auto trait Sync {}
```

### !Send + Sync
Most primitive types we can think of are both `Send` and `Sync`, so let's go over some examples that do not satisfy at least one of them.
When a type is not `Send` but `Sync`, it's safe to share references across threads while giving away ownership is unsafe. 

A good example is the `std::sync::MutexGuard`.
```rust
impl<T: ?Sized> !Send for std::sync::MutexGuard<'_, T>
impl<T: ?Sized + Sync> Sync for std::sync::MutexGuard<'_, T>
```

`MutexGuard` is not `Send` because certain operating systems require mutex locks to be released on the exact thread that acquired them. But this doesn't prohibit the type from being `Sync` since giving away shared references is totally fine as long as the ownership stays in the same thread to release the lock. `T: Sync` is required since `&T` can be earned from `&MutexGuard` through `Deref`.

### Send + !Sync
It might sound a bit weird that a type is `Send` but not `Sync`, but we already saw some examples in the [previous post](../aliasing-mutability-containers-and-pointers). We talked about how `Cell` provides interior mutability through shared references, and that it was only safe in a single-thread context. If we can share references of `Cell` across threads, multiple threads can mutate the inner value, breaking the safety invariant. However, it's fine to pass over the `Cell` itself to another thread as long as the inner value is `Send` since the single-thread invariant still stands.
The same logic can be applied to `RefCell` as well.

```rust
impl<T> Send for Cell<T>
where
    T: Send + ?Sized,
impl<T> Send for RefCell<T>
where
    T: Send + ?Sized,
impl<T> !Sync for Cell<T>
where
    T: ?Sized,
impl<T> !Sync for RefCell<T>
where
    T: ?Sized,
```

### !Send + !Sync
As we went over `Cell` and `RefCell`, you've probably noticed how `Rc` satisfies none of these properties.
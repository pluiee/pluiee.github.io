+++
title = "Making Sense of Send and Sync in Rust"
date = 2026-04-18
[taxonomies]
categories = ["Rust"]
+++
It's common to encounter `Send + Sync` bounds when working with various Rust codebases.
But what do they actually mean, and how do they work under the hood?
When exactly is a type `Send` or `Sync`, and when is it not?

### Send
A type is `Send` if it is safe to send, or transfer ownership to another thread.
```rust
pub unsafe auto trait Send {}
```
Note that implementing `Send` doesn't provide any special methods or behavior; rather, it acts as a marker indicating that the type satisfies a specific safety condition.
The `auto` keyword means the compiler will automatically implement `Send` for a type if all of its inner fields are also `Send`.

Thus, it's quite rare (and requires `unsafe`) to manually implement `Send`. 
Doing so means you're explicitly telling the compiler that a type is safe to send across threads, even if it contains non-`Send` fields.
(Conversely, explicitly opting out with `!Send` is also rare but useful when you want to enforce thread-locality.)

### Sync
A type `T` is `Sync` if and only if its shared reference `&T` is `Send`. 
In other words, a type is `Sync` if it is safe to share references across threads. It's an auto marker trait like `Send`.
```rust
pub unsafe auto trait Sync {}
```

### !Send + Sync
Most primitive types we can think of are both `Send` and `Sync`, so let's explore some examples that lack at least one of these properties.
If a type is `Sync` but not `Send`, it means you can safely share references to it across threads, but you cannot transfer its ownership to another thread.

A classic example is `std::sync::MutexGuard`.
```rust
impl<T: ?Sized> !Send for std::sync::MutexGuard<'_, T>
impl<T: ?Sized + Sync> Sync for std::sync::MutexGuard<'_, T>
```

`MutexGuard` is not `Send` because some operating systems require mutex locks to be released by the exact same thread that acquired them. 
However, this doesn't prevent the type from being `Sync`.
Sharing references is perfectly fine as long as the ownership (and thus the responsibility to release the lock) remains on the original thread. 
The `T: Sync` bound is required because you can obtain a `&T` from a `&MutexGuard` via `Deref`.

### Send + !Sync
It might sound a bit weird for a type to be `Send` but not `Sync`, but we already saw some examples of this in the [previous post](../aliasing-mutability-containers-and-pointers). 
We talked about how `Cell` provides interior mutability through shared references, and how that is only safe in a single-thread context. 
If we can share references of `Cell` across threads, multiple threads could concurrently mutate the inner value, breaking the safety invariant. 

However, transferring ownership of the `Cell` itself to another thread is completely fine (provided the inner value is `Send`), because it guarantees that only one thread can access it at a time. 
The same logic applies to `RefCell` as well.

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
Given our discussion on `Cell` and `RefCell`, you might wonder about `Rc`. 
`Rc` satisfies neither property because its internal reference counter is not atomic.

If `Rc` were `Sync`, multiple threads could borrow it and call `clone()` concurrently, leading to data races on the internal reference count. 
If `Rc` were `Send`, you could clone an `Rc` and send the cloned instance to another thread, which would again allow concurrent, non-atomic modifications to the shared reference count.
```rust
impl<T, A> !Send for Rc<T, A>
where
    A: Allocator,
    T: ?Sized,

impl<T, A> !Sync for Rc<T, A>
where
    A: Allocator,
    T: ?Sized,
```

### Reference
This post is heavily based on Jon Gjentset's [Crust of Rust — Send, Sync, and their implementors](https://youtu.be/yOezcP-XaIw?si=QTW_gd_N1ZrT0INI) session. 
Code snippets are taken from the official std documents.
If you are interested in this topic, watching the original video is highly recommended.
+++
title = "Aliasing, Mutability, Containers and Pointers"
date = 2026-04-17
[taxonomies]
categories = ["Rust"]
+++
### T, &mut T, &T
In Rust, any arbitrary type `T` can either be owned as `T` itself, or used in two forms of aliases: `&mut T` and `&T`.

- `T`: the owned value itself
- `&mut T`: a mutable reference with write access to `T`
- `&T`: an immutable reference that allows reading but not modification

### Aliasing NAND Mutability
Rust's borrow checker enforces a rule known as _Aliasing NAND Mutability_ to guarantee safety at compile time. That is:

- While a `&mut T` exists (Mutability = 1), no other reference to `T` may exist (Aliasing = 0).
- While a `&T` exists (Aliasing = 1), no `&mut T` may exist (Mutability = 0).

### Containers & Pointers
In practice, however, strictly adhering to these constraints is not always straightforward. There are cases where mutation is needed — and can be proven safe — even when multiple aliases exist, or where shared references bound to a specific scope and lifetime make it difficult to express data that is jointly owned and accessed from multiple places.

To accommodate these scenarios while extending the safety guarantees in a flexible way, Rust provides a variety of containers and pointers. Each of them can be examined from two angles:

- In what situations is it useful?
- How does it still uphold safety?

### Cell
`Cell<T>` is a container that enables interior mutability in a single-threaded context — allowing multiple references to coexist while still permitting mutation of the inner `T`. To support this, it never exposes a reference to the inner `T` directly. Let us look at a minimal implementation:
```rust
use std::cell::UnsafeCell;

pub struct Cell<T> {
    value: UnsafeCell<T>,
}

// implied by UnsafeCell
// impl<T> !Sync for Cell<T> {}

impl<T> Cell<T> {
    pub fn new(value: T) -> Self {
        Cell {
            value: UnsafeCell::new(value),
        }
    }

    pub fn set(&self, value: T) {
        // SAFETY: we know no-one else is concurrently mutating self.value (because !Sync)
        // SAFETY: we know we're not invalidating any references, because we never give any out
        unsafe { *self.value.get() = value };
    }

    pub fn get(&self) -> T
    where
        T: Copy,
    {
        // SAFETY: we know no-one else is modifying this value, since only this thread can mutate
        // (because !Sync), and it is executing this function instead.
        unsafe { *self.value.get() }
    }
}
```
`Cell::get` returns a fresh copy of the inner value rather than a reference to it, ensuring that `T` is never exposed externally. As a result, set can replace the inner value without any risk of dangling references.

### RefCell
`RefCell<T>` takes a different approach: it shifts the responsibility of borrow checking from compile time to runtime. This is useful in situations where the compiler cannot statically prove the absence of aliased mutable references, but the developer can guarantee that at any given point at most one mutable reference exists — graph traversal being a common example.
To achieve this, `RefCell` internally tracks references through a runtime state counter. Below is a simplified implementation:

```rust
use crate::cell::Cell;
use std::cell::UnsafeCell;

#[derive(Copy, Clone)]
enum RefState {
    Unshared,
    Shared(usize),
    Exclusive,
}

pub struct RefCell<T> {
    value: UnsafeCell<T>,
    state: Cell<RefState>,
}

// implied by UnsafeCell
// impl<T> !Sync for RefCell<T> {}

impl<T> RefCell<T> {
    pub fn new(value: T) -> Self {
        Self {
            value: UnsafeCell::new(value),
            state: Cell::new(RefState::Unshared),
        }
    }

    pub fn borrow(&self) -> Option<Ref<'_, T>> {
        match self.state.get() {
            RefState::Unshared => {
                self.state.set(RefState::Shared(1));
                Some(Ref { refcell: self })
            }
            RefState::Shared(n) => {
                self.state.set(RefState::Shared(n + 1));
                Some(Ref { refcell: self })
            }
            RefState::Exclusive => None,
        }
    }

    pub fn borrow_mut(&self) -> Option<RefMut<'_, T>> {
        if let RefState::Unshared = self.state.get() {
            self.state.set(RefState::Exclusive);
            // SAFETY: no other references have been given out since state would be
            // Shared or Exclusive.
            Some(RefMut { refcell: self })
        } else {
            None
        }
    }
}

pub struct Ref<'refcell, T> {
    refcell: &'refcell RefCell<T>,
}

impl<T> std::ops::Deref for Ref<'_, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        // SAFETY
        // a Ref is only created if no exclusive references have been given out.
        // once it is given out, state is set to Shared, so no exclusive references are given out.
        // so dereferencing into a shared reference is fine.
        unsafe { &*self.refcell.value.get() }
    }
}

impl<T> Drop for Ref<'_, T> {
    fn drop(&mut self) {
        match self.refcell.state.get() {
            RefState::Exclusive | RefState::Unshared => unreachable!(),
            RefState::Shared(1) => {
                self.refcell.state.set(RefState::Unshared);
            }
            RefState::Shared(n) => {
                self.refcell.state.set(RefState::Shared(n - 1));
            }
        }
    }
}

pub struct RefMut<'refcell, T> {
    refcell: &'refcell RefCell<T>,
}

impl<T> std::ops::Deref for RefMut<'_, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        // SAFETY
        // see safety for DerefMut
        unsafe { &*self.refcell.value.get() }
    }
}

impl<T> std::ops::DerefMut for RefMut<'_, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        // SAFETY
        // a RefMut is only created if no other references have been given out.
        // once it is given out, state is set to Exclusive, so no future references are given out.
        // so we have an exclusive lease on the inner value, so mutably dereferencing is fine.
        unsafe { &mut *self.refcell.value.get() }
    }
}

impl<T> Drop for RefMut<'_, T> {
    fn drop(&mut self) {
        match self.refcell.state.get() {
            RefState::Shared(_) | RefState::Unshared => unreachable!(),
            RefState::Exclusive => {
                self.refcell.state.set(RefState::Unshared);
            }
        }
    }
}
```

Two aspects are worth highlighting: the introduction of `RefState` to track live references at runtime, and the fact that borrow and borrow_mut return the wrapper types `Ref<T>` and `RefMut<T>` — rather than bare `&T` or `&mut T` — so that `RefState` can be updated correctly via their `Drop` implementations when references go out of scope.

### Rc
`Rc` is a smart pointer that enables shared ownership of its inner data.
Whereas ordinary values have a single owner with a compile-time-managed lifetime, data held inside an `Rc` is heap-allocated and jointly owned, with a runtime reference count determining when the memory is freed. Below is a simplified implementation:
```rust
use crate::cell::Cell;
use std::marker::PhantomData;
use std::ptr::NonNull;

struct RcInner<T> {
    value: T,
    refcount: Cell<usize>,
}

pub struct Rc<T> {
    inner: NonNull<RcInner<T>>,
    _marker: PhantomData<RcInner<T>>,
}

impl<T> Rc<T> {
    pub fn new(v: T) -> Self {
        let inner = Box::new(RcInner {
            value: v,
            refcount: Cell::new(1),
        });

        Rc {
            // SAFETY: Box does not give us a null pointer.
            inner: unsafe { NonNull::new_unchecked(Box::into_raw(inner)) },
            _marker: PhantomData,
        }
    }
}

impl<T> std::ops::Deref for Rc<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        // SAFETY: self.inner is a Box that is only deallocated when the last Rc goes away.
        // we have an Rc, therefore the Box has not been deallocated, so deref is fine.
        &unsafe { self.inner.as_ref() }.value
    }
}

impl<T> Clone for Rc<T> {
    fn clone(&self) -> Self {
        let inner = unsafe { self.inner.as_ref() };
        let c = inner.refcount.get();
        inner.refcount.set(c + 1);
        Rc {
            inner: self.inner,
            _marker: PhantomData,
        }
    }
}

impl<T> Drop for Rc<T> {
    fn drop(&mut self) {
        let inner = unsafe { self.inner.as_ref() };
        let c = inner.refcount.get();
        if c == 1 {
            // SAFETY: we are the _only_ Rc left, and we are being dropped.
            // therefore, after us, there will be no Rc's, and no references to T.
            let _ = unsafe { Box::from_raw(self.inner.as_ptr()) };
        } else {
            // there are other Rcs, so don't drop the Box!
            inner.refcount.set(c - 1);
        }
    }
}
```
Notable here is the use of `Box` for heap allocation, a Cell-managed refcount shared across all `Rc` instances, and a `Drop` implementation that deallocates the inner data only when the reference count reaches zero.

### Multi-threaded Environments
All of the containers and pointers covered so far are !Sync and intended for single-threaded use. In multi-threaded contexts, their thread-safe counterparts — `Arc`, `RwLock`, `Mutex`, and others — are used instead. These deserve a dedicated discussion and will be covered separately in a future post.

### Summary
- Rust guarantees safety at compile time through the _Aliasing NAND Mutability_ invariant.
- `Cell`, `RefCell`, and `Rc` each work around or reshape this rule by paying different costs: copying, runtime borrow checking, and runtime reference counting, respectively.
- In multi-threaded environments, thread safety must also be considered, which is where `Arc`, `Mutex`, and `RwLock` come in.

### Reference
This post is heavily based on Jon Gjentset's [Crust of Rust — Smart Pointers and Interior Mutability](https://youtu.be/8O0Nt9qY_vo?si=5OqnKTgWkW0-QgQi) session. 
All code snippets appearing in this post are taken from that original session. 
If you are interested in this topic, watching the original video is highly recommended.
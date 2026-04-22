+++
title = "Static vs Dynamic Dispatch in Rust"
date = 2026-04-22
[taxonomies]
categories = ["Rust"]
+++

### Why do we need dispatch?

Say we want to write a function that returns the length of a string. The most straightforward version looks like this:

```rust
fn strlen(s: String) -> usize {
    s.len()
}
```

This works, but it's quite rigid. What if I want to pass a `&str`? Or a `Cow<str>`? I'd have to either convert every argument into a `String` before calling, or write a bunch of overloads like `strlen_str`, `strlen_cow`, and so on.

What we actually want is: "take *anything* that can be viewed as a string, and give me its length." The concrete type shouldn't matter. Only the capability matters. That's what dispatch is for. It lets us write one function over many types, and leaves the details of *how* to resolve the right behavior to the compiler.

### Static dispatch

Here's a version of `strlen` that accepts anything string-like:

```rust
fn strlen(s: impl AsRef<str>) -> usize {
    s.as_ref().len()
}

// All of these work:
// strlen("hello world!");
// strlen(String::from("hello world!"));
// strlen(&some_string);
```

The `impl AsRef<str>` syntax means "any type `T` that implements `AsRef<str>`." It's syntactic sugar for a generic parameter with a trait bound, so the function above is equivalent to:

```rust
fn strlen<T: AsRef<str>>(s: T) -> usize {
    s.as_ref().len()
}
```

So what does the Rust compiler actually do with this? It performs **monomorphization**. 

For every distinct type `T` that we call `strlen` with, the compiler stamps out a separate, specialized copy of the function. Call it with a `&str` and a `String` in the same program, and you end up with two functions in the final binary: one where `T = &str`, another where `T = String`, each with `as_ref` inlined for that specific type.

This is called *static* dispatch because which version of the function to call is decided entirely at compile time. The upside is speed. There's no indirection, and the compiler can inline aggressively. The downside is code bloat (every `T` gets its own copy) and, more importantly, the fact that the type has to be known at compile time.

### When static dispatch isn't enough

Now, suppose we want a collection of things that all implement some trait, and we want to iterate over them:

```rust
trait Greet {
    fn greet(&self);
}

struct English;
struct Korean;

impl Greet for English {
    fn greet(&self) { println!("Hello!"); }
}
impl Greet for Korean {
    fn greet(&self) { println!("안녕!"); }
}

// What type goes here?
let greeters: Vec<???> = vec![English, Korean];
```

We can't write `Vec<impl Greet>` or `Vec<T: Greet>` here. A `Vec` holds elements of a single type, and monomorphization would need to pick one concrete `T`. But `English` and `Korean` are different types, so static dispatch can't help us.

What we want is to say "a vector of some greeters, where each element might be a different concrete type, and I'll figure out which `greet` to call at runtime." That's **dynamic dispatch**, and Rust spells it with the `dyn` keyword:

```rust
let greeters: Vec<dyn Greet> = vec![English, Korean]; // doesn't compile!
```

Except this doesn't compile. The error will say something about `dyn Greet` not having a known size at compile time. Which brings us to the next section.

### Sized, and why dyn Trait isn't

Almost every type in Rust implements the `Sized` marker trait automatically. It just means "the compiler knows how many bytes this takes up." Rust needs this information to put things on the stack, in structs, in `Vec`s, everywhere.

`dyn Greet` is different. It means "some type that implements `Greet`", but we don't know which type, so we can't know the size. `English` might be zero bytes, `Korean` might be 48 bytes, and a third implementer might be huge. `dyn Greet` is **unsized** (`!Sized`).

The fix is to put it behind a pointer, because pointers themselves are always sized:

```rust
let greeters: Vec<Box<dyn Greet>> = vec![
    Box::new(English),
    Box::new(Korean),
];

for g in &greeters {
    g.greet();
}
```

A `&dyn Greet` works too, if you don't need ownership.

So how does the compiler actually make `g.greet()` call the right method? A `Box<dyn Greet>` (or `&dyn Greet`) isn't a normal pointer. It's a **fat pointer**, two words wide:

- A pointer to the data itself (the `English` or `Korean` value on the heap).
- A pointer to a **vtable**, a small static table of function pointers for this trait, generated once per `(type, trait)` pair by the compiler.

When you call `g.greet()`, the compiler emits code that looks up `greet` in the vtable and calls it through that pointer. The actual concrete type is never known to the caller. Only the vtable entries are.

### Object-safe traits

Not every trait can be turned into a `dyn Trait` though. The requirements come directly from how vtables and fat pointers work. Once you've got that picture in your head, the rules basically derive themselves.

**You can't combine arbitrary traits behind one `dyn`.**

Something like `dyn Greet + Clone` isn't allowed in general, because the fat pointer only has room for one vtable pointer. If you want both, you have to define a new trait that requires both and use `dyn` on that:

```rust
trait GreetAndClone: Greet + Clone {}
```

The exception is auto traits like `Send` and `Sync`. They don't have methods, so they don't need a vtable, which is why `dyn Greet + Send` works.

**Associated types must be fixed.** 

If a trait has `type Item;`, the vtable has no way to carry that around, since types aren't runtime values. So you have to pin it down at the `dyn` site:

```rust
dyn Iterator<Item = u32> // ok
dyn Iterator            // not ok
```

**No methods that take `self` by value, and no methods that return `Self`.** 

Taking `self` by value requires knowing the size to move it, which we don't have. Returning `Self` is even worse: the caller has no idea what type came back, so there's nothing useful they could do with it.

**No generic methods.** 

As we've seen earlier, generic methods get monomorphized per type parameter, which would mean a vtable of unbounded size. You'd need one entry per `T` the method could ever be called with, and the compiler has no way of knowing that up front.

### Any escape hatch?
If only some methods of a trait violate these rules, you can opt them out of dynamic dispatch by adding a `where Self: Sized` bound. Those methods then only exist for concrete types, and the rest of the trait remains object-safe. `Iterator` does exactly this. It has a mountain of generic adapter methods (`map`, `filter`, `collect`, ...), but they're all gated on `Self: Sized`, which is why `dyn Iterator<Item = T>` is still supported.

### Summary
- Static dispatch (`impl Trait` or generics) resolves everything at compile time via monomorphization. It's fast, but the type has to be known up front. 
- Dynamic dispatch (`dyn Trait` behind a pointer) uses a fat pointer with a vtable to resolve calls at runtime, which is flexible enough for runtime type choice.
- The object-safety rules for `dyn Trait` are just whatever the vtable + fat-pointer representation can physically support.

### Reference

This post is heavily based on Jon Gjengset's [Crust of Rust: Dispatch and Fat Pointers](https://youtu.be/xcygqF5LVmM?si=cNLVvN0wbIqGNVj1) session. If you're interested in this topic, watching the original video is highly recommended.
+++
title = "Your Code Doesn't Run the Way You Read It"
date = 2026-04-22
[taxonomies]
categories = ["Rust"]
+++
I recently watched a stream on Rust's `std::sync::atomic` module, and honestly, most of it went over my head on the first pass.
But there's one insight from the session I think it's worth sharing even as I'm still wrapping my head around the rest.

### A test that shouldn't fail
Here's a simplified version of a test case from the stream. Ignore the `Ordering` argument for now.
```rust
#[test]
fn at_least_one_is_non_zero() {
    loom::model(|| {
        let x: &'static _ = Box::leak(Box::new(AtomicUsize::new(0)));
        let y: &'static _ = Box::leak(Box::new(AtomicUsize::new(0)));
        let t1 = spawn(move || {
            x.store(1, Ordering::Relaxed);
            y.load(Ordering::Relaxed)
        });
        let t2 = spawn(move || {
            y.store(1, Ordering::Relaxed);
            x.load(Ordering::Relaxed)
        });
        let r1 = t1.join().unwrap();
        let r2 = t2.join().unwrap();
        assert!(r1 == 1 || r2 == 1);
    });
}
```
Two threads, each writing `1` to their own variable and then reading the other's.
Intuitively, at least one of the reads should see `1`, right?
There are only four lines of code across the two threads, and no matter how you interleave them, one thread's `store` must happen before the other thread's `load`.
So `r1 == 0 && r2 == 0` looks impossible.

### But the test fails
Yet `loom` happily finds an execution where both reads return `0`.
How is that possible?

### What's actually going on
The short answer is that `Relaxed` atomic operations provide almost no ordering guarantees across different memory locations.
They say nothing about how operations on different locations are observed by other threads.

In the execution `loom` finds, you can think of it like this: on real hardware, each CPU core has a store buffer.
When `t1` executes `x.store(1)`, the write might sit in `t1`'s store buffer for a while before it's flushed to memory that `t2` can see.
Meanwhile, `t1` goes ahead and does `y.load()`, which reads from the memory where `t2`'s write to `y` hasn't landed yet either.
Symmetrically for `t2`.
Both threads end up reading the initial `0`, because neither store has been made visible to the other thread at the time of the load.

Note that this doesn't mean `r1` and `r2` are *always* both 0—it's just that there's a legitimate execution order that allows it to happen. (And `loom` is a testing tool that searches through all such possible orderings for you.)


### Sequential execution is a fiction
Here's where it gets interesting. Stepping back from the specifics of atomics, what this really illustrates is something more fundamental about how programs execute.

We tend to read code top-to-bottom and imagine it runs that way.
But underneath, your program isn't really a sequence. It's a **dependency graph**.
Nodes are operations, edges are 'this must happen before that.'
Any execution order that respects the edges is a valid execution, and if two nodes have no edge between them, there's simply no fact of the matter about which one runs first.

Most of the time, we don't notice this. The compiler, the CPU, and the language runtime work hard to preserve the illusion of sequential execution.
In single-threaded code, the illusion is near-perfect.
But the cost of maintaining that illusion across threads is enormous, so at the boundary of concurrency, the curtain gets pulled back and the graph shows through.

That's what's happening in our test.
`t1`'s `store` and `t2`'s `load` aren't connected by any edge in the graph. Nothing in the `Relaxed` ordering forces one to happen before the other, and nothing forces their effects to propagate in any particular order.
So the runtime is free to pick any ordering it likes, including the one where both threads miss each other's writes.

### So how do we fix it?
So how do we make the test pass? We add edges to the graph. That's what the `Ordering` parameter is for.
```rust
x.store(1, Ordering::SeqCst);
y.load(Ordering::SeqCst)
```
Swapping `Relaxed` for `SeqCst` (sequentially consistent) is the heaviest hammer in the box.
It says: all `SeqCst` operations across all threads must appear to happen in some single global order that every thread agrees on.
Under this constraint, the "both reads return 0" execution becomes impossible. If there's a global order, then either `x.store` comes before `x.load` or `y.store` comes before `y.load` (or both), and whichever it is, the corresponding load must see `1`.

There are weaker (and cheaper) orderings too: `Acquire`, `Release`, `AcqRel`. 
It's out of this post's scope to unpack what each of them does exactly (and honestly it's out of my brain's scope too).

The general shape is: each ordering level is a different set of edges you're willing to pay for, and picking the right one is about the minimal set of edges your invariants actually require.

### Summary
- **A program isn't the sequential text you read. It's a graph, and concurrency is just where that truth stops being hideable.** `Ordering` is how you declare, precisely and at a cost, which edges of that graph actually matter to you.

- Don't go deep into the lock-free world unless you really have to. The stream was more than enough to scare me off.

### Reference
As always, this post is heavily based on Jon Gjentset's [Crust of Rust — Atomics and Memory Ordering](https://youtu.be/rMGWeSjctlY?si=fHqPRyGcxaBCyXth) session. 
If you are interested in this topic, watching the original video is highly recommended.
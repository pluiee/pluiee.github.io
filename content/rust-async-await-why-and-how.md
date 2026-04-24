+++
title = "Rust's async/await: Why and How"
date = 2026-04-25
[taxonomies]
categories = ["Rust"]
+++

### Why async in the first place

Before touching any Rust-specific detail, it is worth asking why async exists at all.

In synchronous code, when you issue an I/O operation, the thread that issued it simply stops and waits. If the request takes 100ms, that is 100ms of pure waste from the CPU's perspective, since the CPU has nothing to do on behalf of this thread. I/O-bound work like network requests, disk reads, or timers spends most of its time waiting for something external to complete. There is idle time to reclaim in this case.

One obvious solution is to spin up more OS threads and let the kernel interleave them. When one thread blocks on I/O, the kernel schedules another. This works, but it scales poorly. Each OS thread carries a stack measured in megabytes, every context switch pays the cost of a kernel transition, and the kernel's preemptive scheduler has no idea which threads are actually worth running next. For a server juggling thousands of concurrent connections, most of which are idle at any given moment, this is quite a overhead for a lot of threads that spend most of their time asleep.

async is the answer to this problem. Rather than asking the kernel to manage concurrency for us, we use non-blocking I/O APIs and schedule concurrent operations in user space. A single OS thread can host many logical tasks, each suspending itself at I/O boundaries so others can run. Context switches are just function calls. Per-task memory is kilobytes rather than megabytes. This is often called **user-space concurrency**.

There are two broad design axes for user-space concurrency. The first is cooperative vs preemptive scheduling: do tasks voluntarily yield at known points (coroutines), or can the runtime suspend them at any moment (green threads)? The second, within cooperative schedulers, is stackful vs stackless: does each task carry its own stack, or does the compiler rewrite task bodies into plain data structures that carry no stack at all? Rust picked cooperative and stackless, which is what `Future`s are.

### async fn and Future

The `async` keyword on a function is essentially syntactic sugar. These two are roughly equivalent.

```rust
async fn fetch() -> String {
    // ...
}

fn fetch() -> impl Future<Output = String> {
    async {
        // ...
    }
}
```

`Future` is a trait representing a value that is not yet ready but will eventually produce one. The trait itself looks like this.

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

The important thing to notice is that calling an async function does not run its body. It just hands you back a Future, which does nothing until something calls `poll` on it. If you try to use the return value directly, the compiler will nudge you to `.await` it.

### State machines and await

When the compiler sees an `async fn`, it transforms the body into a state machine. Every `.await` inside the function becomes a potential yield point, and any local variable that needs to survive across a yield point is captured as a field in that state.

This is exactly what "stackless" means in stackless coroutine. There is no separate stack allocated for the async function. Instead, the compiler figures out what needs to live across each yield point and bakes it into a struct. The state machine is just a plain value, and it is as big as its largest state.

To make this concrete, consider:

```rust
async fn work() -> u32 {
    let a = step_one().await;
    let b = step_two(a).await;
    a + b
}
```

The compiler generates something roughly like this.

```rust
enum WorkState {
    Start,
    AwaitingOne { fut: StepOneFuture },
    AwaitingTwo { a: u32, fut: StepTwoFuture },
    Done,
}
```

Each `.await` splits the function into another state. The local `a` has to live across the second `.await`, so it gets stored in the `AwaitingTwo` variant.

It is tempting to think `.await` is what "runs" the future. It isn't. `.await` is a yield point, not an execution engine. Desugared, it looks roughly like this.

```rust
loop {
    match fut.poll(cx) {
        Poll::Ready(value) => break value,
        Poll::Pending => yield,
    }
}
```

So `.await` asks the inner future whether it is done. If yes, take the value and move on. If not, propagate `Pending` up to whoever called us, which propagates it up again, and so on. The state machine pauses at whichever await point produced the `Pending`, and resumes from exactly that point next time it is polled.

### The Executor

So where does the propagated `Pending` finally stop?

Async code has to bottom out somewhere. Something has to hold the top-level future, call `poll` on it, and decide what to do when it returns `Pending`. That something is the executor, and in practice the executor is bundled with an async runtime that also provides the low-level I/O building blocks. tokio is by far the most common choice in Rust, though others like async-std and smol exist.

This is also why `main` cannot be `async` directly. Something non-async has to set up the runtime and block on the top-level future. `#[tokio::main]` is a macro that does exactly that for you.

```rust
#[tokio::main]
async fn main() {
    run().await;
}

// roughly expands to:

fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        run().await;
    });
}
```

We won't go deeper into how the executor schedules tasks or wakes them up. The mental model we need from here on is simple. There is a top-level loop that holds our futures, and when we say "yield", this is the thing we yield back to.

### Combining flows: join and select

Async alone is not useful. You need multiple flows to interleave, otherwise there is no idle time to reclaim. `join!` and `select!` are the simplest way to combine futures.

`join!` waits for all of its futures to complete and returns their results together. 

```rust
let (user, posts) = tokio::join!(fetch_user(id), fetch_posts(id));
```

`select!` waits for the first of several futures to complete and lets you branch on which one finished first. Common use cases are timeouts, cancellation, and listening on multiple channels at once.

```rust
tokio::select! {
    result = fetch_user(id) => handle(result),
    _ = tokio::time::sleep(Duration::from_secs(5)) => timeout(),
}
```

Note that both of these combine multiple futures into a single bigger future. From the executor's point of view there is still just one task, and the interleaving happens inside the combined future.

### spawn: handing tasks to the executor

`tokio::spawn` is different. It registers a future as an independent task with the executor.

```rust
let handle = tokio::spawn(async {
    do_something().await
});

// ... do other work ...

let result = handle.await.unwrap();
```

The caller does not have to wait for it. The task lives on its own, gets scheduled independently, and if it panics, the panic is isolated rather than taking down whatever spawned it. You get a `JoinHandle` back that you can `.await` later if you want the result, or drop if you don't care.

tokio's multi-threaded runtime keeps a pool of OS threads, usually one per core, and spawned tasks can be moved between them. That is where actual parallel execution comes from. async by itself gives you concurrency, and parallelism only shows up once the runtime has multiple threads to place tasks on.

Because of this mobility, spawned futures have to be `'static` and `Send`. The task might outlive its caller and might run on a different thread than the one that spawned it, so it cannot borrow anything from the caller's stack.

### Blocking inside async

The executor drives one task at a time on each of its worker threads. When a task hits an await and returns `Pending`, the worker moves on to another task. This only works because tasks cooperate by yielding at await points.

Now consider what happens if you do this.

```rust
async fn handler() {
    std::thread::sleep(Duration::from_secs(5));
    // ...
}
```

`std::thread::sleep` is a synchronous call that asks the OS to park the thread it runs on. There is no await here, so there is no yield. The worker thread sits frozen for five seconds, and every other task that happened to be scheduled on that thread waits with it. 
If you only have a few worker threads, a handful of these can stall a huge chunk of your server.

The same applies to anything that blocks, including heavy computation that never hits an await.

```rust
async fn handler(data: Vec<u8>) -> Hash {
    compute_expensive_hash(&data) // runs for 200ms, no awaits
}
```

No other task on this worker thread makes progress during those 200ms.

The fix is `tokio::spawn_blocking`. tokio keeps a separate thread pool specifically for blocking work, and `spawn_blocking` moves your synchronous operation onto one of those threads. Meanwhile the async worker threads keep serving other tasks.

```rust
async fn handler(data: Vec<u8>) -> Hash {
    tokio::task::spawn_blocking(move || {
        compute_expensive_hash(&data)
    }).await.unwrap()
}
```

### Blocking mutex across await

`std::sync::Mutex` is fine in async code as long as you never hold it across an `.await`. The moment you do, you have a subtle but real deadlock risk.

Consider this.

```rust
async fn handler(state: Arc<std::sync::Mutex<State>>) {
    let guard = state.lock().unwrap();
    some_async_call().await;  // still holding the lock
    guard.modify();
}
```

When `some_async_call().await` returns `Pending`, the worker thread moves on to another task, while this task keeps holding the mutex. If that other task on the same worker tries to lock the same mutex, it calls `std::sync::Mutex::lock`, which is a blocking operation. It does not yield. It parks the thread waiting for the lock.

But the lock is held by our suspended task, which is sitting in the executor's queue waiting to be polled again, which will only happen when the worker thread is free, which will only happen when the lock is released. The worker is stuck waiting for the lock, and the lock is stuck waiting for the worker. Deadlock.

The fix is `tokio::sync::Mutex`. When you call its `.lock().await` and the mutex is held, it yields instead of blocking the thread. The worker is free to run other tasks, including whichever one holds the lock, which eventually releases it and wakes us up.

```rust
async fn handler(state: Arc<tokio::sync::Mutex<State>>) {
    let mut guard = state.lock().await;
    some_async_call().await;
    guard.modify();
}
```

In short, use `std::sync::Mutex` for short critical sections that contain no awaits (it is faster). Use `tokio::sync::Mutex` when the critical section crosses an await point.

### Summary

- async exists to provide user-space concurrency, sidestepping the memory and context-switching overhead of OS threads for I/O-heavy workloads. Rust implements it as cooperative, stackless coroutines, exposed as the `Future` trait.
- `async fn` desugars to a function returning `impl Future`, and the body is compiled into a state machine whose states are the await points.
- `.await` is a yield point. It propagates `Pending` up the chain until it reaches an executor that owns the top-level future. tokio is the standard choice.
- `join!` and `select!` combine futures within a single task. `spawn` promotes a future to an independent task, and the runtime's thread pool is what actually provides parallelism.

### Reference
- [Crust of Rust - async/await](https://youtu.be/ThjvMReOXYM?si=Ozu_mMzJPjHqixw2)
- [Why async Rust?](https://without.boats/blog/why-async-rust/) 

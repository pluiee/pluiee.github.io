+++
title = "Building a Rust Channel from Scratch"
date = 2026-04-19
[taxonomies]
categories = ["Rust"]
+++

Channels are a fundamental concurrency primitive in Rust, allowing threads to communicate safely by passing messages. At a high level, the concept is simple: senders transmit data, and receivers consume it. But what actually happens under the hood? 

In this post, we will demystify the inner workings of channels by implementing a basic Multi-Producer, Single-Consumer (MPSC) channel from scratch. Afterward, we'll briefly explore some of the common channel flavors you might encounter in the wild.

### Sender and Receiver
The basic interface is fairly straightforward. A channel requires a sender and a receiver, each equipped with `send` and `recv` methods respectively.
```rust
pub fn channel<T>() -> (Sender<T>, Receiver<T>) {
    // TODO
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ping_pong() {
        let (mut tx, mut rx) = channel();
        tx.send(42);
        assert_eq!(rx.recv(), Some(42));
    }
}
```

### Mutex
The sender and receiver need to share a queue where senders can push data and the receiver can pop it. We will wrap the inner data with an `Arc` and a `Mutex` to safely share the queue across threads.

Note that using `#[derive(Clone)]` on `Sender` would unnecessarily impose a `T: Clone` bound, so we will implement the `Clone` trait manually.
```rust
pub struct Sender<T> {
    inner: Arc<Inner<T>>,
}

impl<T> Clone for Sender<T> {
    fn clone(&self) -> Self {
        Sender {
            inner: Arc::clone(&self.inner),
        }
    }
}

impl<T> Sender<T> {
    pub fn send(&mut self, t: T) {
        let queue = self.inner.queue.lock().unwrap();
        queue.push_back(t);
    }
}

pub struct Receiver<T> {
    inner: Arc<Inner<T>>,
}

impl<T> Receiver<T> {
    pub fn recv(&mut self) -> Option<T> {
        let queue = self.inner.queue.lock().unwrap();
        queue.pop_front()
    }
}

struct Inner<T> {
    queue: Mutex<VecDeque<T>>,
}

pub fn channel<T>() -> (Sender<T>, Receiver<T>) {
    let inner = Inner {
        queue: Mutex::default(),
    };
    let inner = Arc::new(inner);
    (
        Sender {
            inner: inner.clone(),
        },
        Receiver {
            inner: inner.clone(),
        },
    )
}
```
This looks reasonable, but recv currently returns an `Option<T>` because it immediately returns `None` if the queue is empty.
The typically expected behavior, however, is for the receiver to block and wait until an item becomes available. 
Constantly polling for a new item is inefficient and wastes CPU cycles, so what's a better approach?

### Condvar
This is where `Condvar` comes in. We can pair a condition variable with our mutex to signal the availability of new items.
The receiver can now be efficiently notified when a new item arrives without wasting any resources, as long as the senders emit a notification when they push an item. 
```rust
struct Inner<T> {
    queue: Mutex<VecDeque<T>>,
    available: Condvar
}

impl<T> Sender<T> {
    pub fn send(&mut self, t: T) {
        let queue = self.inner.queue.lock().unwrap();
        queue.push_back(t);
        drop(queue);
        self.inner.available.notify_one();
    }
}

impl<T> Receiver<T> {
    pub fn recv(&mut self) -> T {
        let queue = self.inner.queue.lock().unwrap();
        loop {
            match queue.pop_front() {
                Some(t) => return t,
                None => {
                    queue = self.inner.available.wait(queue).unwrap();
                }
            }
        }

    }
}
```
You might wonder why a loop is necessary in recv if the condition variable wakes up the thread only when an item is added. 
In reality, threads can experience "spurious wakeups" - waking up without a clear reason. 
Thus, wrapping the wait inside a loop ensures we actually have an item before proceeding.

### Dangling Receiver
So far, everything seems fine. Senders send data, and the receiver waits for and receives it.
But what happens if all senders are dropped while the receiver is still waiting for new messages?
```rust
#[test]
fn closed_tx() {
    let (tx, mut rx) = channel::<()>();
    drop(tx);
    // Can this end?
    let _ = rx.recv();
}
```
Our current implementation will block forever in this scenario. To fix this, it would be a good idea to track the number of active senders.

```rust
pub struct Sender<T> {
    shared: Arc<Shared<T>>,
}

impl<T> Clone for Sender<T> {
    fn clone(&self) -> Self {
        let mut inner = self.shared.inner.lock().unwrap();
        inner.senders += 1;
        drop(inner);

        Sender {
            shared: Arc::clone(&self.shared),
        }
    }
}

impl<T> Drop for Sender<T> {
    fn drop(&mut self) {
        let mut inner = self.shared.inner.lock().unwrap();
        inner.senders -= 1;
        let was_last = inner.senders == 0;
        drop(inner);
        if was_last {
            self.shared.available.notify_one();
        }
    }
}

impl<T> Sender<T> {
    pub fn send(&mut self, t: T) {
        let mut inner = self.shared.inner.lock().unwrap();
        inner.queue.push_back(t);
        drop(inner);
        self.shared.available.notify_one();
    }
}

pub struct Receiver<T> {
    shared: Arc<Shared<T>>,
}

impl<T> Receiver<T> {
    pub fn recv(&mut self) -> Option<T> {
        let mut inner = self.shared.inner.lock().unwrap();
        loop {
            match inner.queue.pop_front() {
                Some(t) => return Some(t),
                None if inner.senders == 0 => return None,
                None => {
                    inner = self.shared.available.wait(inner).unwrap();
                }
            }
        }
    }
}

struct Inner<T> {
    queue: VecDeque<T>,
    senders: usize,
}

struct Shared<T> {
    inner: Mutex<Inner<T>>,
    available: Condvar,
}

pub fn channel<T>() -> (Sender<T>, Receiver<T>) {
    let inner = Inner {
        queue: VecDeque::default(),
        senders: 1,
    };
    let shared = Shared {
        inner: Mutex::new(inner),
        available: Condvar::new(),
    };
    let shared = Arc::new(shared);
    (
        Sender {
            shared: shared.clone(),
        },
        Receiver {
            shared: shared.clone(),
        },
    )
}
```

Notice how we now manage the number of senders alongside the queue inside the mutex.
The receiver will return `None` if there are no more senders available instead of blocking indefinitely.

### Single Consumer Optimization
Although our goal was an MPSC channel, notice that the underlying logic for `Sender` and `Receiver` currently doesn't enforce this. 
In fact, our implementation could easily support multiple receivers as well.

How can we take advantage of the single-consumer constraint? Since there's only one receiver, it is perfectly safe to grab all available items from the shared queue at once, rather than taking them one by one. After all, there's no one else to consume the items!

This is a significant optimization: the receiver only needs to acquire the shared mutex when its local buffer is completely empty.
```rust
pub struct Receiver<T> {
    shared: Arc<Shared<T>>,
    buffer: VecDeque<T>,
}

impl<T> Receiver<T> {
    pub fn recv(&mut self) -> Option<T> {
        if let Some(t) = self.buffer.pop_front() {
            return Some(t);
        }

        let mut inner = self.shared.inner.lock().unwrap();
        loop {
            match inner.queue.pop_front() {
                Some(t) => {
                    std::mem::swap(&mut self.buffer, &mut inner.queue);
                    return Some(t);
                }
                None if inner.senders == 0 => return None,
                None => {
                    inner = self.shared.available.wait(inner).unwrap();
                }
            }
        }
    }
}
```
Notice how we swap the entire queue intto the local buffer if an item is available.
Lock contention can be drastically reduced in certain conditions because the mutex is only acquired when this buffer runs out. 

### Alternative Implementations
Our implementation relied on `Mutex`, `Condvar`, and `VecDeque`. 
Production-ready channel implementations might use a `LinkedList` instead of a `VecDeque` to avoid allocation overhead during resizing, or employ lock-free data structures using atomic operations instead of `Mutex` and `Condvar`. 
We won't delve into those advanced techniques in this post.

### Channel Flavors
Before wrapping up, it's worth mentioning the basic flavors of channels that are widely used in Rust.

- **Unbounded Channels**: A channel with infinite capacity. `send()` never blocks. Our implementation is an example of an unbounded channel.
- **Bounded Channels**: A channel with a fixed capacity limit. `send()` will block if the channel is full, providing natural back-pressure.
- **Rendezvous Channels**: A bounded channel with zero capacity. It forces the sender and receiver to meet at the exact same time to hand off data, mostly used for thread synchronization.
- **Oneshot Channels**: A channel designed to send exactly one message.

Channel implementations can support these flavors through distinct types or by modifying the runtime behavior of a single type.

### Reference
This post is heavily based on Jon Gjentset's [Crust of Rust — Channels](https://youtu.be/b4mS5UPHh20?si=3Tjihx_uC4spOhYO) session. 
Code snippets are also taken from the original session.
If you are interested in this topic, watching the original video is highly recommended.
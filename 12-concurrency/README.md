# Stage 12: Fearless Concurrency ⚡

> *Data races are bugs that crash at 3 AM in production. Rust prevents them at compile time. Welcome to fearless concurrency.*

## What You'll Learn

- How threads work in Rust (and the OS)
- Spawning threads and joining them
- Message passing with channels (mpsc)
- Shared state with `Mutex<T>` and `RwLock<T>`
- Atomic types for lock-free programming
- The `Send` and `Sync` marker traits
- Scoped threads
- Rayon for easy data parallelism
- Common concurrency patterns

## Prerequisites

- Completed [Stage 11: Smart Pointers](../11-smart-pointers/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Concurrency vs Parallelism](./01-concurrency-vs-parallelism.md) | Definitions, OS threads, green threads, why it matters |
| 2 | [Spawning Threads](./02-spawning-threads.md) | thread::spawn, JoinHandle, move closures with threads |
| 3 | [Message Passing — Channels](./03-channels.md) | mpsc, Sender, Receiver, multiple producers |
| 4 | [Shared State — Mutex](./04-mutex.md) | Mutex<T>, MutexGuard, poisoning, Arc<Mutex<T>> |
| 5 | [RwLock — Reader-Writer Lock](./05-rwlock.md) | Multiple readers OR one writer, when to prefer over Mutex |
| 6 | [Atomic Types](./06-atomics.md) | AtomicBool, AtomicUsize, Ordering, lock-free patterns |
| 7 | [Send & Sync Traits](./07-send-and-sync.md) | What makes a type thread-safe, auto traits |
| 8 | [Scoped Threads](./08-scoped-threads.md) | thread::scope, borrowing data across threads safely |
| 9 | [Rayon — Easy Parallelism](./09-rayon.md) | par_iter, parallel sorting, work stealing |

## By the End of This Stage

You'll write concurrent Rust code confidently, knowing the compiler has your back against data races.

---

**Previous:** [← Stage 11: Smart Pointers](../11-smart-pointers/) | **Next:** [Stage 13: Async Rust →](../13-async-rust/)

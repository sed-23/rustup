# Stage 13: Async Rust 🌊

> *Handle thousands of connections with a single thread. Async Rust is how you build high-performance servers and apps.*

## What You'll Learn

- What async programming is and why it exists
- Futures — Rust's abstraction for async computation
- The `async`/`await` syntax
- Tokio — the most popular async runtime
- Async I/O (reading files, network requests)
- Streams — async iterators
- `Pin` and `Unpin` — why they exist
- `select!` and `join!` — running futures concurrently
- Common async patterns and pitfalls

## Prerequisites

- Completed [Stage 12: Concurrency](../12-concurrency/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Why Async?](./01-why-async.md) | Blocking vs non-blocking, event loops, OS-level I/O |
| 2 | [Futures Explained](./02-futures.md) | The Future trait, Poll, Pending, Ready |
| 3 | [async/await Syntax](./03-async-await.md) | Writing async functions, .await, async blocks |
| 4 | [Tokio Runtime](./04-tokio-runtime.md) | Setting up Tokio, #[tokio::main], spawning tasks |
| 5 | [Async I/O](./05-async-io.md) | Async file reading, TCP/UDP, reqwest for HTTP |
| 6 | [Streams](./06-streams.md) | Async iterators, StreamExt, processing data streams |
| 7 | [select! & join!](./07-select-and-join.md) | Running multiple futures, racing, timeout patterns |
| 8 | [Pin & Unpin](./08-pin-and-unpin.md) | Self-referential structs, why Pin exists, practical usage |
| 9 | [Async Patterns & Pitfalls](./09-async-patterns.md) | Common mistakes, cancellation, structured concurrency |

## By the End of This Stage

You'll build high-performance async applications with Tokio. You'll understand the async model deeply — not just the syntax.

---

**Previous:** [← Stage 12: Concurrency](../12-concurrency/) | **Next:** [Stage 14: DSA — Arrays, Strings & Hashing →](../14-dsa-arrays-strings-hashing/)

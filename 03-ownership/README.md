# Stage 3: Ownership — Rust's Superpower 🦸

> *This is THE concept that makes Rust unique. No garbage collector. No manual memory management. Just ownership.*

## What You'll Learn

- How computer memory actually works (stack vs heap)
- Rust's ownership rules (there are only 3!)
- What "move semantics" means and why it matters
- How borrowing and references prevent data races at compile time
- The difference between `String` and `&str`
- What slices are and how they work in memory
- Common ownership patterns used in real-world code

## Prerequisites

- Completed [Stage 2: Rust Fundamentals](../02-rust-fundamentals/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Memory 101 — Stack & Heap](./01-memory-stack-and-heap.md) | How your computer stores data, with diagrams |
| 2 | [The Three Rules of Ownership](./02-ownership-rules.md) | Every value has an owner, one owner, drop when out of scope |
| 3 | [Move Semantics](./03-move-semantics.md) | What happens when you assign or pass values |
| 4 | [Clone & Copy](./04-clone-and-copy.md) | Deep copies, the Copy trait, when to use each |
| 5 | [References & Borrowing](./05-references-and-borrowing.md) | Immutable references, the borrow checker |
| 6 | [Mutable References](./06-mutable-references.md) | &mut, the "one writer OR many readers" rule |
| 7 | [Slices](./07-slices.md) | String slices, array slices, how they reference memory |
| 8 | [String vs &str](./08-string-vs-str.md) | Owned vs borrowed strings — when to use which |

## By the End of This Stage

You'll understand Rust's ownership model — the feature that eliminates entire categories of bugs. You'll be able to read and fix borrow checker errors confidently.

---

**Previous:** [← Stage 2: Rust Fundamentals](../02-rust-fundamentals/) | **Next:** [Stage 4: Structuring Data →](../04-structuring-data/)

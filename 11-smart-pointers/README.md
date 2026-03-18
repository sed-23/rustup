# Stage 11: Smart Pointers & The Heap 🧠

> *Go beyond stack-allocated data. Smart pointers give you controlled, safe access to heap memory.*

## What You'll Learn

- What smart pointers are and why they exist
- `Box<T>` — the simplest heap allocation
- `Rc<T>` — reference counted shared ownership
- `Arc<T>` — atomic reference counting for threads
- `RefCell<T>` — interior mutability pattern
- `Cell<T>` — copy-based interior mutability
- `Weak<T>` — breaking reference cycles
- The `Deref` and `Drop` traits
- Memory layout diagrams for every smart pointer

## Prerequisites

- Completed [Stage 10: Modules & Crates](../10-modules-and-crates/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [What Are Smart Pointers?](./01-what-are-smart-pointers.md) | Pointers with metadata + behavior, ownership semantics |
| 2 | [Box<T> — Heap Allocation](./02-box.md) | When to use Box, recursive types, memory layout |
| 3 | [The Deref Trait](./03-deref-trait.md) | Deref coercion, implementing Deref, * operator |
| 4 | [The Drop Trait](./04-drop-trait.md) | Custom cleanup, drop order, std::mem::drop |
| 5 | [Rc<T> — Shared Ownership](./05-rc.md) | Reference counting, Rc::clone, when to use |
| 6 | [RefCell<T> — Interior Mutability](./06-refcell.md) | Runtime borrow checking, Rc<RefCell<T>> pattern |
| 7 | [Cell<T> & Copy Semantics](./07-cell.md) | Interior mutability for Copy types |
| 8 | [Weak<T> — Breaking Cycles](./08-weak.md) | Preventing memory leaks, weak references |
| 9 | [Arc<T> — Thread-Safe Sharing](./09-arc.md) | Atomic reference counting, Arc<Mutex<T>> pattern |

## By the End of This Stage

You'll understand every smart pointer in Rust's standard library, when to use each one, and exactly how they manage memory.

---

**Previous:** [← Stage 10: Modules & Crates](../10-modules-and-crates/) | **Next:** [Stage 12: Concurrency →](../12-concurrency/)

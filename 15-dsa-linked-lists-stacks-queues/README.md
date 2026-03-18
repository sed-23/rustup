# Stage 15: DSA — Linked Lists, Stacks & Queues 🔗

> *Linked lists in Rust are notoriously tricky because of ownership. This is where you truly understand what Rust demands — and why.*

## What You'll Learn

- Why linked lists are hard in Rust (ownership + pointers)
- Singly linked lists with `Box<T>`
- Doubly linked lists with `Rc<RefCell<T>>`
- How the standard library's `LinkedList` works
- Stack implementation (LIFO) — Vec-backed and linked
- Queue implementation (FIFO) — VecDeque-backed
- Monotonic stack and monotonic queue patterns
- When to use (and NOT use) linked lists in Rust
- Memory layout comparisons (array vs linked)

## Prerequisites

- Completed [Stage 14: DSA — Arrays, Strings & Hashing](../14-dsa-arrays-strings-hashing/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Why Linked Lists Are Hard in Rust](./01-why-linked-lists-are-hard.md) | Ownership vs pointers, the fundamental tension |
| 2 | [Singly Linked List](./02-singly-linked-list.md) | Building a linked list with Box, push, pop, iter |
| 3 | [Doubly Linked List](./03-doubly-linked-list.md) | Rc<RefCell<T>> approach, Weak for prev pointers |
| 4 | [Unsafe Linked List](./04-unsafe-linked-list.md) | Raw pointers approach, when unsafe is justified |
| 5 | [Stack — LIFO](./05-stack.md) | Vec-backed stack, linked stack, real-world uses |
| 6 | [Queue — FIFO](./06-queue.md) | VecDeque-backed queue, circular buffer concepts |
| 7 | [Monotonic Stack & Queue](./07-monotonic-stack-queue.md) | Next greater element, sliding window max |
| 8 | [Practice Problems](./08-practice-problems.md) | 15+ problems: reverse list, merge lists, valid parentheses |

## By the End of This Stage

You'll understand the deep relationship between data structures and Rust's ownership model. You'll implement linked lists, stacks, and queues from scratch.

---

**Previous:** [← Stage 14: DSA — Arrays & Hashing](../14-dsa-arrays-strings-hashing/) | **Next:** [Stage 16: DSA — Trees & Graphs →](../16-dsa-trees-and-graphs/)

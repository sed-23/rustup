# Stage 6: Closures & Iterators 🔄

> *Functional programming meets systems programming. Closures and iterators make Rust code elegant AND fast.*

## What You'll Learn

- What closures are and how they capture their environment
- The three closure traits: `Fn`, `FnMut`, `FnOnce`
- How closures relate to ownership and borrowing
- The iterator trait and how it works
- Iterator adaptors (map, filter, take, skip, zip, chain, etc.)
- Iterator consumers (collect, sum, count, fold, etc.)
- Lazy evaluation — why iterators are zero-cost
- Building your own iterator

## Prerequisites

- Completed [Stage 5: Collections](../05-collections/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Closures — Anonymous Functions](./01-closures.md) | Syntax, type inference, capturing variables |
| 2 | [Fn, FnMut, FnOnce](./02-closure-traits.md) | The three closure traits, when each is used |
| 3 | [Closures & Ownership](./03-closures-and-ownership.md) | move closures, capturing by reference vs value |
| 4 | [The Iterator Trait](./04-iterator-trait.md) | How iteration works, the next() method |
| 5 | [Iterator Adaptors](./05-iterator-adaptors.md) | map, filter, enumerate, zip, chain, take, skip, peekable |
| 6 | [Iterator Consumers](./06-iterator-consumers.md) | collect, sum, count, fold, any, all, find, position |
| 7 | [Building Custom Iterators](./07-custom-iterators.md) | Implementing Iterator for your own types |

## By the End of This Stage

You'll write clean, functional-style Rust code using closures and iterator chains. You'll understand why Rust's iterators are "zero-cost abstractions" — as fast as hand-written loops.

---

**Previous:** [← Stage 5: Collections](../05-collections/) | **Next:** [Stage 7: Error Handling →](../07-error-handling/)

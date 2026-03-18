# Stage 9: Lifetimes Demystified ⏳

> *The most feared topic in Rust — made simple. Lifetimes just answer: "How long does this reference live?"*

## What You'll Learn

- Why lifetimes exist (preventing dangling references)
- Lifetime annotation syntax (`'a`)
- How the compiler infers lifetimes (elision rules)
- Lifetimes in function signatures
- Lifetimes in struct definitions
- The `'static` lifetime
- Common lifetime patterns and how to think about them

## Prerequisites

- Completed [Stage 8: Generics & Traits](../08-generics-and-traits/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Why Lifetimes Exist](./01-why-lifetimes.md) | The dangling reference problem, what lifetimes solve |
| 2 | [Lifetime Annotation Syntax](./02-lifetime-syntax.md) | 'a, 'b — what the annotations mean |
| 3 | [Lifetimes in Functions](./03-lifetimes-in-functions.md) | When and how to annotate function signatures |
| 4 | [Lifetime Elision Rules](./04-lifetime-elision.md) | The 3 rules the compiler uses to infer lifetimes |
| 5 | [Lifetimes in Structs](./05-lifetimes-in-structs.md) | Structs that hold references |
| 6 | [The 'static Lifetime](./06-static-lifetime.md) | What 'static means, string literals, owned data |
| 7 | [Lifetime Patterns & Tips](./07-lifetime-patterns.md) | Common patterns, when to restructure instead of annotate |

## By the End of This Stage

Lifetimes won't scare you anymore. You'll read lifetime annotations fluently and know when (and when NOT) to add them.

---

**Previous:** [← Stage 8: Generics & Traits](../08-generics-and-traits/) | **Next:** [Stage 10: Modules & Crates →](../10-modules-and-crates/)

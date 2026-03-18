# Stage 21: Macros — Metaprogramming 🪄

> *Write code that writes code. Macros are Rust's most powerful metaprogramming tool.*

## What You'll Learn

- What macros are and why Rust has them
- Declarative macros with `macro_rules!`
- How macro patterns and repetitions work
- Procedural macros — the three kinds
- Custom derive macros (like `#[derive(Debug)]`)
- Attribute macros (like `#[tokio::main]`)
- Function-like procedural macros
- Building a real-world derive macro from scratch

## Prerequisites

- Completed [Stage 20: Unsafe Rust & FFI](../20-unsafe-and-ffi/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Why Macros?](./01-why-macros.md) | Code generation, DRY, what macros can do that functions can't |
| 2 | [Declarative Macros (macro_rules!)](./02-macro-rules.md) | Pattern matching, repetitions, common patterns |
| 3 | [macro_rules! Advanced Patterns](./03-macro-rules-advanced.md) | TT munchers, internal rules, debugging macros |
| 4 | [Procedural Macros Overview](./04-proc-macros-overview.md) | TokenStream, syn, quote, the proc-macro crate |
| 5 | [Derive Macros](./05-derive-macros.md) | Building a custom #[derive(MyTrait)] from scratch |
| 6 | [Attribute Macros](./06-attribute-macros.md) | Custom #[my_attribute] macros |
| 7 | [Function-like Proc Macros](./07-function-like-macros.md) | my_macro!(...) that generates arbitrary code |

## By the End of This Stage

You'll read and write Rust macros confidently. You'll build your own derive macro and understand the proc-macro ecosystem.

---

**Previous:** [← Stage 20: Unsafe Rust & FFI](../20-unsafe-and-ffi/) | **Next:** [Stage 22: Testing & Docs →](../22-testing-and-docs/)

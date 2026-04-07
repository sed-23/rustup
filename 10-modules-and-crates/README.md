# Stage 10: Modules & Project Organization 📁

> *Small programs fit in one file. Real programs need structure. Learn to organize Rust projects like a pro.*

## The Module System Mental Model

Every Rust project is a **tree of modules** rooted at the crate root (`main.rs` or `lib.rs`). Think of it like a folder structure, but enforced by the compiler:

```
crate (root)
├── config          mod config;
│   └── defaults    mod defaults;
├── users           mod users;
│   ├── model       mod model;
│   └── handlers    mod handlers;
└── error           mod error;
```

Each `mod` declaration is like opening a folder. Items inside are private by default — you explicitly `pub` the ones you want to expose. This forces you to design a real public API rather than accidentally exposing internals.

The `use` keyword is how you bring items into scope so you don't have to type full paths everywhere. And `pub use` lets you re-export items, giving users a clean flat API even when your internals are deeply nested.

## Key Concepts at a Glance

| Concept | One-line explanation |
|---------|----------------------|
| **crate** | The unit of compilation — one `src/lib.rs` or `src/main.rs` |
| **package** | A `Cargo.toml` with one or more crates |
| **mod** | Declares a module (namespace + privacy boundary) |
| **pub** | Makes an item visible outside its module |
| **use** | Brings a path into scope (shortcut for long paths) |
| **pub use** | Re-exports an item — builds a clean public API |
| **workspace** | Multiple packages sharing one `Cargo.lock` and `target/` |

## What You'll Learn

- Rust's module system (`mod`, `pub`, `use`)
- Packages, crates, and the crate root
- Visibility rules and encapsulation
- How to split code across multiple files
- Workspaces for multi-crate projects
- How to publish your own crate to crates.io
- Real-world project organization patterns

## Prerequisites

- Completed [Stage 9: Lifetimes](../09-lifetimes/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Packages & Crates](./01-packages-and-crates.md) | Binary vs library crates, Cargo.toml anatomy |
| 2 | [Defining Modules](./02-defining-modules.md) | mod keyword, nested modules, module tree |
| 3 | [Paths & Visibility](./03-paths-and-visibility.md) | pub, pub(crate), pub(super), absolute & relative paths |
| 4 | [The use Keyword](./04-use-keyword.md) | Bringing paths into scope, re-exporting, aliases |
| 5 | [Splitting Into Files](./05-splitting-into-files.md) | Moving modules to separate files and directories |
| 6 | [Cargo Workspaces](./06-workspaces.md) | Multi-crate projects, shared dependencies |
| 7 | [Publishing to crates.io](./07-publishing-crates.md) | Documentation, metadata, versioning, publishing |

## By the End of This Stage

You'll organize Rust projects like a professional. Multi-file, multi-module, multi-crate — you'll handle it all.

---

**Previous:** [← Stage 9: Lifetimes](../09-lifetimes/) | **Next:** [Stage 11: Smart Pointers →](../11-smart-pointers/)

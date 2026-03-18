# 🦀 Rustup — From Zero to Hero in Rust

> **The most comprehensive, beginner-friendly Rust tutorial series on the internet.**
> Every concept explained like you're five. Memory diagrams included. No prior experience required.

![Rust](https://img.shields.io/badge/Language-Rust-orange?style=for-the-badge&logo=rust)
![Tutorials](https://img.shields.io/badge/Tutorials-195+-blue?style=for-the-badge)
![Stages](https://img.shields.io/badge/Stages-25-green?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)

---

## 🎯 What is this?

This is a **complete roadmap** to mastering Rust — from writing your very first line of code to building production-ready applications. Each tutorial includes:

- 📖 **Crystal-clear explanations** (no jargon without definitions)
- 🧠 **Memory diagrams** (see exactly what happens on the stack & heap)
- 💻 **Runnable code examples** (copy, paste, learn)
- 🏭 **Real-world context** (how professionals actually use this)
- ⚠️ **Common mistakes** (and how to avoid them)
- 🏋️ **Exercises** (practice makes permanent)
- 🔗 **What's next** (clear learning path)

---

## 🗺️ Learning Path

### 🟢 Beginner (Stages 1–7)
*Build your foundation. Understand what makes Rust special.*

| Stage | Title | Tutorials | Description |
|:-----:|-------|:---------:|-------------|
| 1 | [**The Starting Line**](./01-the-starting-line/) | 6 | Why Rust exists, install it, write Hello World, learn Cargo |
| 2 | [**Rust Fundamentals**](./02-rust-fundamentals/) | 8 | Variables, types, functions, control flow — the building blocks |
| 3 | [**Ownership — Rust's Superpower**](./03-ownership/) | 8 | THE concept that makes Rust unique. Stack, heap, borrowing |
| 4 | [**Structuring Data**](./04-structuring-data/) | 8 | Structs, enums, Option, Result, pattern matching |
| 5 | [**Collections Deep Dive**](./05-collections/) | 8 | Vec, String, HashMap and friends — with memory layouts |
| 6 | [**Closures & Iterators**](./06-closures-and-iterators/) | 7 | Functional programming in Rust, iterator chains |
| 7 | [**Error Handling Mastery**](./07-error-handling/) | 7 | panic!, Result, ?, custom errors, real-world patterns |

### 🟡 Intermediate (Stages 8–13)
*Level up. Understand Rust's powerful type system and concurrency model.*

| Stage | Title | Tutorials | Description |
|:-----:|-------|:---------:|-------------|
| 8 | [**Generics & Traits**](./08-generics-and-traits/) | 9 | Write flexible, reusable code with zero runtime cost |
| 9 | [**Lifetimes Demystified**](./09-lifetimes/) | 7 | The infamous lifetime annotations — made simple |
| 10 | [**Modules & Project Organization**](./10-modules-and-crates/) | 7 | Structure real projects, publish crates, workspaces |
| 11 | [**Smart Pointers & The Heap**](./11-smart-pointers/) | 9 | Box, Rc, Arc, RefCell — advanced memory management |
| 12 | [**Fearless Concurrency**](./12-concurrency/) | 9 | Threads, channels, Mutex — safe parallelism |
| 13 | [**Async Rust**](./13-async-rust/) | 9 | Futures, Tokio, async/await — modern async programming |

### 🔴 DSA in Rust (Stages 14–18)
*Data Structures & Algorithms implemented in Rust with memory visualizations.*

| Stage | Title | Tutorials | Description |
|:-----:|-------|:---------:|-------------|
| 14 | [**Arrays, Strings & Hashing**](./14-dsa-arrays-strings-hashing/) | 8 | Array tricks, string algorithms, hash map internals |
| 15 | [**Linked Lists, Stacks & Queues**](./15-dsa-linked-lists-stacks-queues/) | 8 | Classic data structures vs Rust's ownership model |
| 16 | [**Trees & Graphs**](./16-dsa-trees-and-graphs/) | 9 | Binary trees, BST, graphs, BFS, DFS, shortest paths |
| 17 | [**Sorting, Searching & Recursion**](./17-dsa-sorting-searching/) | 9 | Every sorting algorithm, binary search, backtracking |
| 18 | [**Dynamic Programming & Greedy**](./18-dsa-dp-and-greedy/) | 8 | Memoization, tabulation, classic DP, greedy strategies |

### ⚫ Advanced (Stages 19–22)
*Master the dark arts. Push Rust to its limits.*

| Stage | Title | Tutorials | Description |
|:-----:|-------|:---------:|-------------|
| 19 | [**Advanced Type System**](./19-advanced-types/) | 8 | Newtype, phantom data, type-state, advanced traits |
| 20 | [**Unsafe Rust & FFI**](./20-unsafe-and-ffi/) | 8 | Raw pointers, unsafe blocks, calling C, interop |
| 21 | [**Macros — Metaprogramming**](./21-macros/) | 7 | macro_rules!, proc macros, derive macros |
| 22 | [**Testing & Documentation**](./22-testing-and-docs/) | 8 | Unit/integration/doc tests, benchmarks, coverage |

### 🌟 Real-World Projects (Stages 23–25)
*Build real things. Ship real code.*

| Stage | Title | Tutorials | Description |
|:-----:|-------|:---------:|-------------|
| 23 | [**Project: CLI Tool**](./23-project-cli/) | 6 | Build a complete command-line application |
| 24 | [**Project: Web API**](./24-project-web-api/) | 7 | REST API with Axum, database, auth, deployment |
| 25 | [**Ecosystem & Idiomatic Rust**](./25-ecosystem-and-patterns/) | 7 | Design patterns, crates, performance, CI/CD |

---

## 🚀 How to Use This

1. **Start at Stage 1** — even if you know other languages
2. **Read each tutorial in order** — they build on each other
3. **Type every code example** — don't just read, DO
4. **Complete the exercises** — they cement your understanding
5. **Build the projects** — stages 23-25 tie everything together

### Prerequisites

- A computer (Windows, macOS, or Linux)
- Basic understanding of what programming is (variables, loops, functions)
- That's it. Seriously.

---

## 📁 Repository Structure

```
rustup/
├── README.md                              ← You are here
├── 01-the-starting-line/
│   ├── README.md                          ← Stage overview
│   ├── 01-why-rust.md
│   ├── 02-installing-rust.md
│   └── ...
├── 02-rust-fundamentals/
│   ├── README.md
│   ├── 01-variables-and-mutability.md
│   └── ...
├── ...
└── 25-ecosystem-and-patterns/
    ├── README.md
    └── ...
```

Each stage folder contains:
- `README.md` — Stage overview, learning objectives, prerequisites
- Numbered tutorial files in reading order
- Code examples inline (copy-paste ready)

---

## 🤝 Contributing

Found a typo? Want to improve an explanation? PRs are welcome!

1. Fork this repo
2. Create a branch (`git checkout -b fix/typo-in-stage-3`)
3. Commit your changes
4. Open a Pull Request

---

## 📜 License

This project is licensed under the MIT License — see the [LICENSE](./LICENSE) file.

---

## ⭐ Star This Repo

If this helps you learn Rust, give it a ⭐! It helps others find it too.

---

<p align="center">
  <i>Built with ❤️ for the Rust community</i><br>
  <i>"The language empowering everyone to build reliable and efficient software."</i>
</p>

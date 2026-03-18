# Stage 20: Unsafe Rust & FFI ⚠️

> *Sometimes you need to step outside Rust's safety guarantees. Unsafe Rust is a scalpel — precise, powerful, and dangerous if misused.*

## What You'll Learn

- What `unsafe` means (and what it does NOT mean)
- The five unsafe superpowers
- Raw pointers (`*const T`, `*mut T`)
- Unsafe functions and methods
- Unsafe traits
- Foreign Function Interface (FFI)
- Calling C code from Rust
- Calling Rust code from C and Python
- When unsafe is justified (and when it isn't)

## Prerequisites

- Completed [Stage 19: Advanced Types](../19-advanced-types/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [What is Unsafe Rust?](./01-what-is-unsafe.md) | The 5 superpowers, unsafe blocks, trust boundary |
| 2 | [Raw Pointers](./02-raw-pointers.md) | *const T, *mut T, dereferencing, pointer arithmetic |
| 3 | [Unsafe Functions & Methods](./03-unsafe-functions.md) | Writing and calling unsafe functions, safe wrappers |
| 4 | [Unsafe Traits](./04-unsafe-traits.md) | When trait implementation requires unsafe guarantees |
| 5 | [FFI — Basics](./05-ffi-basics.md) | extern "C", #[no_mangle], ABI, linking |
| 6 | [Calling C from Rust](./06-calling-c-from-rust.md) | bindgen, libc, wrapping C libraries safely |
| 7 | [Calling Rust from C/Python](./07-calling-rust-from-others.md) | cdylib, PyO3, exposing Rust to other languages |
| 8 | [Unsafe Best Practices](./08-unsafe-best-practices.md) | Minimizing unsafe, safe abstractions, auditing |

## By the End of This Stage

You'll know when and how to use unsafe Rust responsibly. You'll interop with C libraries and expose Rust to other languages.

---

**Previous:** [← Stage 19: Advanced Types](../19-advanced-types/) | **Next:** [Stage 21: Macros →](../21-macros/)

# Rust Playground & Tools 🛠️

> **Rust has some of the best developer tools in any language. Let's explore them all — from the online playground to formatters and linters.**

---

## Table of Contents

- [The Rust Playground](#the-rust-playground)
- [rustfmt — The Code Formatter](#rustfmt--the-code-formatter)
- [Clippy — The Linter](#clippy--the-linter)
- [rust-analyzer — Your IDE Superpower](#rust-analyzer--your-ide-superpower)
- [rustdoc — Documentation Generator](#rustdoc--documentation-generator)
- [Offline Documentation](#offline-documentation)
- [Other Useful Tools](#other-useful-tools)
- [Your Development Workflow](#your-development-workflow)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Rust Playground

The Rust Playground is a **website where you can write, compile, and run Rust code** — with zero installation.

### 🔗 [play.rust-lang.org](https://play.rust-lang.org)

### When to Use the Playground

| Use Case | Example |
|----------|---------|
| Quick experiments | "What does this code do?" |
| Sharing code | Send someone a link to your code |
| Testing small ideas | "Will this compile?" |
| Learning | Try examples from this tutorial |
| No Rust installed | Using someone else's computer |

### Playground Features

#### 1. Run Code

Type code in the editor and click **"Run"** (or press `Ctrl+Enter`):

```rust
fn main() {
    let x = 5;
    let y = 10;
    println!("{} + {} = {}", x, y, x + y);
}
```

#### 2. Choose Rust Edition & Channel

At the top, you can switch between:
- **Stable / Beta / Nightly** — different release channels
- **2015 / 2018 / 2021 / 2024** — different editions

#### 3. Share Your Code

Click the **"Share"** button to get a permalink. Anyone can click it and see your exact code. This is invaluable for:
- Asking for help online
- Showing someone a bug
- Sharing examples

#### 4. Tools Menu

The **"Tools"** dropdown offers:
- **Rustfmt** — format your code
- **Clippy** — lint your code
- **Miri** — detect undefined behavior (advanced)
- **Expand macros** — see what macros generate

#### 5. Assembly / LLVM IR / MIR / WASM

Under the **"..."** menu, you can see:
- **ASM** — the actual x86 assembly your code compiles to
- **LLVM IR** — the intermediate representation
- **MIR** — Rust's mid-level IR
- **WASM** — WebAssembly output

This is fascinating for understanding performance, but you don't need it as a beginner.

### Playground Limitations

- Can't read files from disk
- Can't make network requests
- Limited execution time (about 10 seconds)
- Limited to a whitelist of popular crates
- Single file only (no multi-file projects)

---

## rustfmt — The Code Formatter

`rustfmt` automatically formats your Rust code to follow the **official Rust style guidelines**.

### Why Use a Formatter?

```
Without formatter:
  - Everyone formats differently
  - Pull requests have formatting noise
  - Debates about style waste time
  - Code is harder to read

With formatter:
  - EVERYONE's code looks the same
  - No formatting debates
  - Code reviews focus on logic, not style
  - Consistent readability across all Rust projects
```

### How to Use

#### Command Line

```bash
# Format all files in your project
cargo fmt

# Check if formatting is correct (without changing files)
cargo fmt -- --check
```

#### In VS Code

If you followed the editor setup from Tutorial 2, code is automatically formatted on save!

### Example: Before and After

**Before (your messy code):**
```rust
fn main(){let x=5;
let y  =    10;
    println!(  "{}+{}={}"  ,x,y,x+y)  ;
        if x>3{println!("big");}
}
```

**After `cargo fmt`:**
```rust
fn main() {
    let x = 5;
    let y = 10;
    println!("{} + {} = {}", x, y, x + y);
    if x > 3 {
        println!("big");
    }
}
```

Consistent spacing, indentation, and brace placement — automatically.

### Customizing rustfmt

Create a `rustfmt.toml` file in your project root:

```toml
# rustfmt.toml
max_width = 100                    # Maximum line width (default: 100)
tab_spaces = 4                     # Spaces per indent (default: 4)
edition = "2021"                   # Rust edition
use_small_heuristics = "Default"   # How to handle short items
```

Most people use the defaults. The Rust community strongly values **consistent style across all projects**.

> **Real-World Practice:** Most professional Rust projects enforce `cargo fmt --check` in their CI/CD pipeline. If your code isn't formatted, the build fails. This means ZERO time wasted on style discussions.

---

## Clippy — The Linter

Clippy is Rust's **official linter** — it analyzes your code and suggests improvements.

### What's a Linter?

A linter is a tool that analyzes your code for:
- Common mistakes
- Performance issues
- Style problems
- Potential bugs
- Unidiomatic patterns

Think of Clippy as a code review from an experienced Rust developer — automated and instant.

#### The History of Linters

The term "lint" comes from a tool written by **Stephen C. Johnson** at Bell Labs in 1978 for the C programming language. Just as a lint roller picks up small fibers from clothing, the `lint` program picked up small errors from C code.

The original `lint` caught common C bugs like:
- Calling a function with the wrong number of arguments (C didn't check this!)
- Using a variable before initialization
- Unreachable code

Over time, linting became standard practice in every language: ESLint (JavaScript), Pylint/flake8 (Python), RuboCop (Ruby), golint (Go). Rust's Clippy was created in 2015 by **Manish Goregaokar** and named after Microsoft's infamous Office Assistant. The irony is intentional — unlike the annoying Clippy paperclip that offered useless advice, Rust's Clippy provides genuinely helpful suggestions.

Clippy currently has **over 700 lints** and is considered one of the most comprehensive linters in any language ecosystem.

### How to Use

```bash
# Run Clippy on your project
cargo clippy

# Run with warnings treated as errors (useful for CI)
cargo clippy -- -D warnings
```

### Example: Clippy in Action

**Your code:**
```rust
fn main() {
    let x = 3.14;
    let y = 0;
    
    if y == 0 {
        println!("zero");
    } else if y == 0 {
        println!("also zero");  // Unreachable!
    }
    
    let v = vec![1, 2, 3];
    if v.len() == 0 {  // Clippy will suggest a better way
        println!("empty");
    }
    
    let s = "hello".to_string();
    let s2 = s.clone();  // Unnecessary clone
    println!("{}", s2);
}
```

**Clippy output:**
```
warning: this `if` has the same condition as a previous `if`
 --> src/main.rs:6:7
  |
6 |     } else if y == 0 {
  |               ^^^^^^
  
warning: length comparison to zero
  --> src/main.rs:11:8
   |
11 |     if v.len() == 0 {
   |        ^^^^^^^^^^^^^ help: consider using `is_empty()`: `v.is_empty()`

warning: redundant clone
  --> src/main.rs:15:17
   |
15 |     let s2 = s.clone();
   |                 ^^^^^^^^ help: remove this
```

Clippy tells you:
1. You have a duplicate condition (likely a bug)
2. Use `is_empty()` instead of `len() == 0` (more idiomatic)
3. The `clone()` is unnecessary (the original isn't used after)

### Clippy Lint Categories

| Category | What It Catches |
|----------|----------------|
| **correctness** | Actual bugs and errors |
| **suspicious** | Code that's likely wrong |
| **style** | Non-idiomatic code |
| **complexity** | Unnecessarily complicated code |
| **perf** | Performance issues |
| **pedantic** | Extra strict (off by default) |
| **nursery** | Experimental lints (off by default) |

### Silencing a Lint

Sometimes Clippy is wrong (rare, but it happens):

```rust
#[allow(clippy::needless_return)]
fn my_function() -> i32 {
    return 42;  // Clippy would normally say "remove the return keyword"
}
```

> **Real-World Practice:** Most Rust teams run `cargo clippy` in CI/CD, often with `-- -D warnings` to treat all warnings as errors. This keeps code quality high automatically.

---

## rust-analyzer — Your IDE Superpower

`rust-analyzer` is the **Language Server Protocol (LSP)** implementation for Rust. It powers the Rust experience in VS Code, Neovim, Helix, and other editors.

### What is LSP? — A Revolution in Editor Tooling

Before LSP, every editor needed its own plugin for every language. Vim needed a separate Vim-Python plugin, a Vim-Java plugin, etc. VS Code needed separate TypeScript support, separate C++ support, etc. This was an $M \times N$ problem: $M$ editors times $N$ languages = $M \times N$ separate implementations.

**The Language Server Protocol (LSP)** was invented by Microsoft in 2016 (originally for TypeScript in VS Code). The idea: languages implement a **single server** that speaks a standard protocol, and editors implement a **single client**. Now it's an $M + N$ problem instead of $M \times N$:

```
Before LSP:                         After LSP:
  Editor A ──── Language 1          Editor A ──┐
  Editor A ──── Language 2          Editor B ──┤── LSP ──┬── Language Server 1
  Editor B ──── Language 1          Editor C ──┘         ├── Language Server 2
  Editor B ──── Language 2                               └── Language Server 3
  (M × N connectors)                (M + N connectors)
```

`rust-analyzer` is Rust's LSP server. It was created by **Aleksey Kladov** (matklad) starting in 2018 as a replacement for the older RLS (Rust Language Server). rust-analyzer is arguably the best LSP implementation of any language — it's fast, accurate, and feature-rich.

### What Does It Do?

| Feature | What It Looks Like |
|---------|-------------------|
| **Autocomplete** | Type `v.` and see all methods available on a vector |
| **Type hints** | See inferred types inline, even when not written |
| **Error highlighting** | Red squiggles before you compile |
| **Go to definition** | Ctrl+Click any function to see its source code |
| **Find references** | Right-click → "Find All References" |
| **Hover docs** | Hover over any function/type to see documentation |
| **Rename symbol** | Rename a variable everywhere it's used |
| **Code actions** | Lightbulb suggestions for fixes |
| **Inline type hints** | Shows types like `let x: i32 = 5` even when you write `let x = 5` |

### Type Hints — Seeing the Invisible

One of rust-analyzer's best features is **inline type hints**. Rust has type inference, meaning you often don't write types explicitly:

```rust
let x = 5;            // What type is x? rust-analyzer shows: i32
let y = 3.14;         // Shows: f64
let v = vec![1, 2];   // Shows: Vec<i32>
let s = "hello";      // Shows: &str
```

In your editor, you'll see ghost-like text showing the types. This makes learning Rust MUCH easier.

### Useful Keyboard Shortcuts (VS Code)

| Shortcut | Action |
|----------|--------|
| `Ctrl+Click` | Go to definition |
| `F12` | Go to definition |
| `Alt+F12` | Peek definition (inline popup) |
| `Shift+F12` | Find all references |
| `F2` | Rename symbol |
| `Ctrl+.` | Quick fixes / code actions |
| `Ctrl+Space` | Trigger autocomplete |
| `Ctrl+Shift+P` | Command palette → "Rust Analyzer:" commands |

### Useful rust-analyzer Commands

Open the command palette (`Ctrl+Shift+P`) and type "Rust Analyzer":

- **Rust Analyzer: Expand Macro Recursively** — see what a macro expands to
- **Rust Analyzer: View Memory Layout** — see the size and layout of a type
- **Rust Analyzer: Interpret Function** — evaluate a function at IDE time
- **Rust Analyzer: Restart Server** — if things get stuck

---

## rustdoc — Documentation Generator

Rust has built-in documentation generation. You write special comments, and `rustdoc` generates beautiful HTML docs.

### Writing Documentation Comments

```rust
/// Adds two numbers together.
///
/// # Examples
///
/// ```
/// let result = add(2, 3);
/// assert_eq!(result, 5);
/// ```
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

The `///` comments are **documentation comments**. They support Markdown formatting.

### Generating Documentation

```bash
# Generate docs for your project
cargo doc

# Generate and open in browser
cargo doc --open

# Include documentation for dependencies too
cargo doc --document-private-items
```

### docs.rs

Every crate published to crates.io automatically gets documentation generated at [docs.rs](https://docs.rs/).

For example:
- Standard library: [doc.rust-lang.org/std](https://doc.rust-lang.org/std/)
- Serde: [docs.rs/serde](https://docs.rs/serde)
- Tokio: [docs.rs/tokio](https://docs.rs/tokio)

---

## Offline Documentation

Rust installs complete documentation locally. Access it anytime, even without internet:

```bash
rustup doc               # Opens the Rust book + std library docs
rustup doc --std          # Opens just the standard library docs
rustup doc --book         # Opens "The Rust Programming Language" book
rustup doc --reference    # Opens the Rust Reference
```

This is incredibly useful on flights, trains, or anywhere without internet.

---

## Other Useful Tools

### cargo-watch — Auto-Rebuild on Save

Tired of manually running `cargo check` after every change?

```bash
# Install
cargo install cargo-watch

# Auto-run check on file save
cargo-watch -x check

# Auto-run tests on file save
cargo-watch -x test

# Auto-run program on file save
cargo-watch -x run
```

### cargo-edit — Better Dependency Management

(Built into Cargo since Rust 1.62, but the external version has more features)

```bash
cargo install cargo-edit

cargo add serde            # Add a dependency
cargo add serde --features derive  # With features
cargo rm serde             # Remove a dependency
cargo upgrade              # Upgrade all dependencies
```

### cargo-expand — See Macro Expansion

```bash
cargo install cargo-expand

# See what macros expand to
cargo expand
```

This shows you what `println!`, `vec![]`, `#[derive(Debug)]`, and other macros actually generate. Invaluable for understanding macros.

### sccache — Shared Compilation Cache

If you work on multiple Rust projects, `sccache` caches compiled artifacts across projects:

```bash
cargo install sccache

# Set in your environment (add to shell config)
export RUSTC_WRAPPER=sccache
```

This can dramatically speed up builds when dependencies overlap between projects.

### bacon — More Beautiful cargo-watch

```bash
cargo install bacon

# Run in your project
bacon        # Shows errors/warnings with a better UI than cargo-watch
```

---

## Your Development Workflow

Here's the recommended workflow as a Rust beginner:

### Setup (Once)

1. Install Rust via rustup
2. Install VS Code + rust-analyzer
3. Enable format on save
4. Run clippy on check (settings from Tutorial 2)

### Daily Workflow

```
1. Create/open project:  cargo new my_project && cd my_project
2. Open in editor:       code .
3. Write code in src/main.rs
4. Check for errors:     Look at VS Code squiggles, or run cargo check
5. Run your program:     cargo run
6. Fix errors:           Read the compiler messages, fix, repeat
7. Format code:          Happens on save (or cargo fmt)
8. Lint code:            cargo clippy
9. Commit:               git add . && git commit -m "description"
```

### When You're Stuck

1. **Read the error message** — Rust errors are helpful, read the WHOLE thing
2. **Check the docs** — `rustup doc --std` or docs.rs
3. **Try the Playground** — isolate the problem in a small example
4. **Search** — "rust [error code]" usually finds the answer
5. **Ask** — Rust community is helpful:
   - [users.rust-lang.org](https://users.rust-lang.org/) — official forum
   - [r/rust](https://reddit.com/r/rust) — Reddit
   - [Rust Discord](https://discord.gg/rust-lang) — real-time chat
   - Stack Overflow with the `[rust]` tag

---

## Exercises

### Exercise 1: Playground Exploration

1. Go to [play.rust-lang.org](https://play.rust-lang.org)
2. Write a program that prints "Hello from the Playground!"
3. Click "Share" and save the link
4. Try the "Tools → Clippy" option on this code:
   ```rust
   fn main() {
       let v = vec![1, 2, 3];
       if v.len() > 0 {
           println!("Not empty");
       }
   }
   ```
   What does Clippy suggest?

### Exercise 2: Format Some Messy Code

Create a file with intentionally messy formatting:

```rust
fn main(){let x=5;let y=10;if x>3{println!("Big: {}",
x);}else{println!("Small");}let v=vec![1,2,3,4,5];for i in &v{println!("{}",i);}}
```

Run `cargo fmt` and see the transformation.

### Exercise 3: Clippy Suggestions

Write this code and run `cargo clippy`:

```rust
fn main() {
    let mut x = 5;
    let y = x;
    
    let name = "Alice".to_string();
    let greeting = format!("{}{}", "Hello, ", name);
    println!("{}", greeting);
    
    let v: Vec<i32> = Vec::new();
    if v.len() == 0 {
        println!("empty");
    }
}
```

How many suggestions does Clippy give? Fix each one.

### Exercise 4: rust-analyzer Exploration

1. Open a Rust project in VS Code
2. Type `let v = vec![1, 2, 3];` — can you see the type hint?
3. Type `v.` and browse the autocomplete suggestions — how many methods does Vec have?
4. Ctrl+Click on `vec!` — where does it take you?
5. Hover over `println!` — what docs appear?

---

## Summary

| Tool | Purpose | Command |
|------|---------|---------|
| **Rust Playground** | Run Rust code in the browser | [play.rust-lang.org](https://play.rust-lang.org) |
| **rustfmt** | Auto-format code to official style | `cargo fmt` |
| **Clippy** | Lint and suggest improvements | `cargo clippy` |
| **rust-analyzer** | IDE features (autocomplete, errors, etc.) | VS Code extension |
| **rustdoc** | Generate HTML documentation | `cargo doc --open` |
| **Offline docs** | Browse docs without internet | `rustup doc` |
| **cargo-watch** | Auto-rebuild on file save | `cargo-watch -x check` |
| **cargo-expand** | See macro expansions | `cargo expand` |

### Key Takeaway

> Rust's tooling is world-class. `cargo fmt` eliminates style debates, `clippy` catches mistakes before code review, and `rust-analyzer` makes your editor smarter than you. These tools work together to make Rust development a joy.

---

## What's Next?

You've got the full toolkit. Let's wrap up Stage 1 by seeing how Rust compares to languages you might already know.

**Next Tutorial:** [Rust vs Other Languages →](./06-rust-vs-other-languages.md)

---

<p align="center">
  <i>Tutorial 5 of 6 — Stage 1: The Starting Line</i>
</p>

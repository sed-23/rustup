# Understanding Cargo 📦

> **Cargo is Rust's build system AND package manager. It's often called the best part of the Rust ecosystem — and for good reason.**

---

## Table of Contents

- [What is Cargo?](#what-is-cargo)
- [Creating a Project with Cargo](#creating-a-project-with-cargo)
- [The Cargo.toml File — Your Project's ID Card](#the-cargotoml-file--your-projects-id-card)
- [The src/ Directory](#the-src-directory)
- [The target/ Directory](#the-target-directory)
- [Cargo Commands — Your Daily Toolkit](#cargo-commands--your-daily-toolkit)
- [Adding Dependencies (Using Other People's Code)](#adding-dependencies-using-other-peoples-code)
- [What is crates.io?](#what-is-cratesio)
- [The Cargo.lock File](#the-cargolock-file)
- [Cargo Workspaces (Preview)](#cargo-workspaces-preview)
- [Real-World Cargo.toml Examples](#real-world-cargotoml-examples)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What is Cargo?

Cargo does **everything** related to building your Rust project:

| What | How Cargo Helps |
|------|----------------|
| **Create projects** | `cargo new` creates the folder structure for you |
| **Compile code** | `cargo build` calls the compiler with the right flags |
| **Run code** | `cargo run` builds and runs in one step |
| **Manage dependencies** | Add libraries to `Cargo.toml` and Cargo downloads them |
| **Run tests** | `cargo test` finds and runs all your tests |
| **Generate docs** | `cargo doc` generates HTML documentation |
| **Publish packages** | `cargo publish` uploads your crate to crates.io |
| **Check code** | `cargo check` verifies without fully compiling (faster) |
| **Format code** | `cargo fmt` auto-formats your code |
| **Lint code** | `cargo clippy` suggests improvements |

### How Does Cargo Compare to Other Package Managers?

| Language | Package Manager | Build Tool | Cargo Equivalent |
|----------|----------------|------------|-----------------|
| JavaScript | npm / yarn / pnpm | webpack / vite | Cargo does BOTH |
| Python | pip | setuptools / poetry | Cargo does BOTH |
| Java | Maven / Gradle | Maven / Gradle | Cargo does BOTH |
| C/C++ | vcpkg / conan | CMake / Make | Cargo does BOTH |
| Go | go modules | go build | Very similar to Cargo |

In most languages, you need **separate tools** for package management and building. Cargo handles **everything in one tool**.

---

## Creating a Project with Cargo

### Binary Project (Program You Can Run)

```bash
cargo new my_project
```

This creates:

```
my_project/
  ├── Cargo.toml       ← Project configuration
  ├── .git/            ← Git repository (auto-created!)
  ├── .gitignore       ← Ignores target/ directory
  └── src/
        └── main.rs    ← Your code starts here
```

### Library Project (Code Other People Can Use)

```bash
cargo new my_library --lib
```

This creates:

```
my_library/
  ├── Cargo.toml
  ├── .git/
  ├── .gitignore
  └── src/
        └── lib.rs     ← Library root (instead of main.rs)
```

### What's the Difference?

| Type | Entry File | Purpose | Has `fn main()`? |
|------|-----------|---------|------------------|
| **Binary** (default) | `src/main.rs` | A program you run | Yes |
| **Library** (`--lib`) | `src/lib.rs` | Code others import and use | No |

You can have BOTH in one project — more on that in Stage 10.

### Naming Conventions

```bash
# Cargo project names use snake_case (underscores)
cargo new my_cool_project    ✅
cargo new my-cool-project    ✅ (hyphens also work, converted to underscores internally)
cargo new MyCoolProject      ⚠️ (works but not conventional)
```

---

## The Cargo.toml File — Your Project's ID Card

Open the `Cargo.toml` that was generated:

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"

[dependencies]
```

Let's break down every line:

### `[package]` Section

```toml
[package]                # This section describes YOUR project
name = "my_project"      # The name of your project/crate
version = "0.1.0"        # Semantic versioning: MAJOR.MINOR.PATCH
edition = "2021"         # Which Rust edition to use
```

#### What is `edition`?

Rust releases new **editions** every 3 years to introduce breaking syntax changes:

| Edition | Year | Notable Changes |
|---------|------|----------------|
| 2015 | 2015 | First edition |
| 2018 | 2018 | Async/await, module system improvements |
| 2021 | 2021 | Closures, IntoIterator for arrays, or patterns |
| 2024 | 2024 | Latest edition |

> Always use the latest edition for new projects. Cargo sets this automatically. Old editions still compile — Rust is backwards compatible.

#### Semantic Versioning

```
version = "0.1.0"
             │ │ │
             │ │ └── PATCH: Bug fixes (backwards compatible)
             │ └──── MINOR: New features (backwards compatible)
             └────── MAJOR: Breaking changes
```

- `0.x.y` means "this is still in initial development" (pre-1.0)
- `1.0.0` means "this is stable, we promise not to break things"
- `0.1.0` → `0.1.1` = bug fix
- `0.1.0` → `0.2.0` = new feature
- `0.1.0` → `1.0.0` = stable release!

### `[dependencies]` Section

This is where you list external libraries your project uses. It starts empty:

```toml
[dependencies]
```

We'll add dependencies shortly.

### More Fields You Can Add

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <you@example.com>"]   # Who wrote this
description = "A short description"          # Shows on crates.io
license = "MIT"                              # The license
repository = "https://github.com/you/repo"  # Source code link
readme = "README.md"                         # Path to README
keywords = ["cli", "tool"]                   # Search keywords
categories = ["command-line-utilities"]       # Category on crates.io
```

You don't need all of these for development — they're mainly used when publishing to crates.io.

---

## The src/ Directory

All your Rust source code lives in `src/`:

```
src/
  ├── main.rs     ← Entry point for binary crates
  ├── lib.rs      ← Entry point for library crates (if you have one)
  └── ...         ← Other modules/files (we'll learn about these in Stage 10)
```

### Why Everything Goes in src/

Cargo follows a **convention over configuration** philosophy:

```
✅ Source code → src/
✅ Tests → tests/ (integration tests) or inline (unit tests)
✅ Benchmarks → benches/
✅ Examples → examples/
✅ Build scripts → build.rs (project root)
```

You don't configure these paths — Cargo knows where to look. This means every Rust project has the same structure, making it easy to navigate any Rust codebase.

---

## The target/ Directory

After you run `cargo build`, a `target/` directory appears:

```
target/
  ├── debug/                    ← Debug builds go here
  │     ├── my_project          ← Your compiled binary
  │     ├── my_project.d        ← Dependency info
  │     ├── deps/               ← Compiled dependencies
  │     ├── build/              ← Build script output
  │     └── incremental/        ← Incremental compilation cache
  └── release/                  ← Release builds go here (cargo build --release)
        └── my_project          ← Optimized binary
```

### Important Facts About target/

1. **It's BIG** — `target/` can easily be hundreds of MB or even GB for large projects. Dependencies get compiled here.
2. **It's in .gitignore** — Cargo auto-creates a `.gitignore` that excludes `target/`. Never commit it to git.
3. **It's safe to delete** — `rm -rf target/` (or `Remove-Item -Recurse target`) is safe. Cargo will rebuild everything next time.
4. **It's cached** — Cargo uses incremental compilation. After the first build, subsequent builds are much faster because only changed code is recompiled.

### Cleaning Up

```bash
cargo clean    # Deletes the entire target/ directory
```

---

## Cargo Commands — Your Daily Toolkit

Here's every Cargo command you'll use, from most frequent to least:

### Commands You'll Use Every Day

```bash
cargo run                    # Build + run your project
cargo check                  # Quick error check (no binary produced)
cargo build                  # Compile the project
cargo test                   # Run all tests
cargo fmt                    # Auto-format your code
cargo clippy                 # Lint your code (suggests improvements)
```

### Commands You'll Use Occasionally

```bash
cargo build --release        # Optimized release build
cargo doc --open             # Generate & open documentation in browser
cargo update                 # Update dependencies to latest compatible versions
cargo clean                  # Delete the target/ directory
cargo add <crate>            # Add a dependency (Rust 1.62+)
cargo remove <crate>         # Remove a dependency
```

### Commands for Publishing

```bash
cargo login                  # Authenticate with crates.io
cargo publish                # Upload your crate to crates.io
cargo install <crate>        # Install a binary crate globally
```

### The Most Important Command

If there's one command to remember, it's:

```bash
cargo run
```

This builds your project (if needed) and runs it. You'll type this hundreds of times.

---

## Adding Dependencies (Using Other People's Code)

One of Cargo's best features is how easy it is to use external libraries.

### Method 1: Edit Cargo.toml Directly

```toml
[dependencies]
rand = "0.8"           # Random number generation
serde = "1.0"          # Serialization/deserialization
```

### Method 2: Use `cargo add` (Recommended)

```bash
cargo add rand
```

This automatically adds the latest version of `rand` to your `Cargo.toml`:

```toml
[dependencies]
rand = "0.8.5"
```

### Using a Dependency in Your Code

```rust
// src/main.rs
use rand::Rng;  // Import the Rng trait from the rand crate

fn main() {
    let mut rng = rand::thread_rng();
    let secret_number: u32 = rng.gen_range(1..=100);
    println!("Random number: {}", secret_number);
}
```

When you run `cargo build` or `cargo run`, Cargo:

1. Reads `Cargo.toml` to find dependencies
2. Downloads the source code from crates.io (first time only)
3. Compiles the dependency
4. Links it with your code
5. Caches everything in `target/`

### Version Syntax

```toml
[dependencies]
# Exact version
serde = "=1.0.193"          # ONLY version 1.0.193

# Caret (default) — compatible updates
serde = "1.0"               # Any 1.x.y where x >= 0  (same as "^1.0")
serde = "1.0.193"           # Any 1.0.y where y >= 193 (same as "^1.0.193")
serde = "0.2.3"             # Any 0.2.y where y >= 3   (pre-1.0 is stricter)

# Tilde — more restrictive
serde = "~1.0.193"          # Any 1.0.y where y >= 193

# Wildcard
serde = "1.*"               # Any 1.x.y

# From git
my_crate = { git = "https://github.com/user/repo" }

# From local path
my_crate = { path = "../my_crate" }
```

For most cases, just use the default: `crate_name = "MAJOR.MINOR"`.

---

## What is crates.io?

[crates.io](https://crates.io) is Rust's official package registry — like npm for JavaScript or PyPI for Python.

### Key Facts

- Over **140,000+** crates available
- Completely free to use and publish to
- Immutable — once a version is published, it can never be changed or deleted (ensures reproducible builds)
- Every crate has a page with documentation, source link, and download stats

### Finding Crates

| Method | URL |
|--------|-----|
| **crates.io** | [https://crates.io](https://crates.io) |
| **lib.rs** | [https://lib.rs](https://lib.rs) — better search and categorization |
| **docs.rs** | [https://docs.rs](https://docs.rs) — auto-generated docs for every crate |

### Popular Crates You'll Use

| Crate | Purpose | Downloads |
|-------|---------|-----------|
| `serde` | Serialization (JSON, TOML, etc.) | Billions |
| `tokio` | Async runtime | Billions |
| `rand` | Random numbers | Hundreds of millions |
| `clap` | CLI argument parsing | Hundreds of millions |
| `reqwest` | HTTP client | Hundreds of millions |
| `axum` | Web framework | Tens of millions |
| `tracing` | Logging/tracing | Hundreds of millions |
| `anyhow` | Error handling (applications) | Hundreds of millions |
| `thiserror` | Error handling (libraries) | Hundreds of millions |

---

## The Cargo.lock File

After your first build, you'll notice a `Cargo.lock` file appeared:

```
my_project/
  ├── Cargo.toml     ← What you want: "I need serde ~1.0"
  ├── Cargo.lock     ← What you got: "I'm using serde 1.0.193 exactly"
  └── src/
        └── main.rs
```

### What is Cargo.lock?

- `Cargo.toml` = "I want version 1.0 or compatible"
- `Cargo.lock` = "On this exact build, I used version 1.0.193"

It records the **exact versions** of every dependency that was used.

### Should You Commit Cargo.lock to Git?

| Project Type | Commit Cargo.lock? | Why |
|-------------|-------------------|-----|
| **Binary/App** | ✅ YES | Everyone building your app gets the same exact versions |
| **Library** | ❌ NO | Let the consumer's project determine versions |

### Updating Locked Versions

```bash
cargo update              # Update ALL dependencies to latest compatible versions
cargo update -p serde     # Update only the serde crate
```

---

## Cargo Workspaces (Preview)

For large projects with multiple related crates, Cargo supports **workspaces**:

```
my_workspace/
  ├── Cargo.toml          ← Workspace root
  ├── crate_a/
  │     ├── Cargo.toml
  │     └── src/
  ├── crate_b/
  │     ├── Cargo.toml
  │     └── src/
  └── crate_c/
        ├── Cargo.toml
        └── src/
```

Workspaces share a single `target/` directory and `Cargo.lock`, so dependencies are compiled once. We'll cover this in depth in Stage 10.

---

## Real-World Cargo.toml Examples

### A CLI Tool

```toml
[package]
name = "my-grep"
version = "0.1.0"
edition = "2021"
description = "A simplified grep clone"
license = "MIT"

[dependencies]
clap = { version = "4", features = ["derive"] }   # CLI argument parsing
regex = "1"                                         # Regular expressions
colored = "2"                                       # Colored terminal output
anyhow = "1"                                        # Easy error handling

[dev-dependencies]
assert_cmd = "2"    # For testing CLI commands
predicates = "3"    # Assertions for tests
```

### A Web Server

```toml
[package]
name = "my-api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"                                         # Web framework
tokio = { version = "1", features = ["full"] }       # Async runtime
serde = { version = "1", features = ["derive"] }     # Serialization
serde_json = "1"                                      # JSON support
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio"] }
tower-http = { version = "0.5", features = ["cors"] } # HTTP middleware
tracing = "0.1"                                       # Logging
tracing-subscriber = "0.3"                            # Log output
```

Notice the `features` syntax:

```toml
tokio = { version = "1", features = ["full"] }
```

**Features** are optional functionality in a crate. You only enable what you need, keeping your binary smaller. Think of it like installing a program and choosing which components to install.

---

## Exercises

### Exercise 1: Create Your First Cargo Project

```bash
cargo new my_first_project
cd my_first_project
cargo run
```

Verify you see `Hello, world!`.

### Exercise 2: Modify and Rebuild

Edit `src/main.rs` to print your name. Run `cargo run` again. Notice how Cargo only recompiles changed files (incremental compilation).

### Exercise 3: Add a Dependency

```bash
cargo add rand
```

Then write a program that generates and prints a random number between 1 and 100:

```rust
use rand::Rng;

fn main() {
    let number = rand::thread_rng().gen_range(1..=100);
    println!("Your lucky number is: {}", number);
}
```

Run it several times and watch the number change.

### Exercise 4: Explore Cargo Commands

Run each of these and observe what they do:

```bash
cargo check        # How long does it take?
cargo build        # How long does it take?
cargo build        # Run again - how long now? (hint: incremental!)
cargo clean        # What disappeared?
cargo build        # How long now that we cleaned?
```

### Exercise 5: Explore target/

Run `cargo build`, then explore the `target/debug/` directory. Find your compiled binary. Run it directly (not through Cargo).

---

## Summary

| Concept | Detail |
|---------|--------|
| **Cargo** | Rust's build tool + package manager (all-in-one) |
| **Cargo.toml** | Project configuration (name, version, dependencies) |
| **Cargo.lock** | Exact versions used (auto-generated) |
| **src/** | All source code goes here |
| **target/** | Compiled output (never commit to git) |
| **cargo run** | Build + run (most used command) |
| **cargo check** | Fast error checking without full compilation |
| **cargo add** | Add a dependency from crates.io |
| **crates.io** | Rust's package registry (140,000+ crates) |
| **features** | Optional functionality in crates |
| **edition** | Rust syntax version (2015, 2018, 2021, 2024) |

### Key Takeaway

> Cargo is one of Rust's biggest strengths. It handles project creation, compilation, dependencies, testing, documentation, and publishing — all in one tool. Once you learn Cargo, you never want to go back to manual build systems.

---

## What's Next?

You know how to create projects and manage dependencies with Cargo. Let's explore some other tools that make Rust development even smoother.

**Next Tutorial:** [Rust Playground & Tools →](./05-playground-and-tools.md)

---

<p align="center">
  <i>Tutorial 4 of 6 — Stage 1: The Starting Line</i>
</p>

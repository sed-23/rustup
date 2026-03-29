# thiserror & anyhow — Error Handling Crates 🎯

> **"Write error types in 10 lines instead of 50 — thiserror handles the boilerplate, anyhow handles the rest."**

---

## Table of Contents

- [The Boilerplate Problem](#the-boilerplate-problem)
- [thiserror — For Library Authors](#thiserror--for-library-authors)
- [thiserror Deep Dive](#thiserror-deep-dive)
- [anyhow — For Application Authors](#anyhow--for-application-authors)
- [anyhow Deep Dive](#anyhow-deep-dive)
- [thiserror vs anyhow — When to Use Which](#thiserror-vs-anyhow--when-to-use-which)
- [Other Error Crates](#other-error-crates)
- [The Rust Error Handling Ecosystem Evolution](#the-rust-error-handling-ecosystem-evolution)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Boilerplate Problem

In the previous tutorial, we built custom error types by hand. Let's look at what that required for a fairly simple enum with three variants:

```rust
use std::fmt;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
    Config(String),
}

// Implement Display — one arm per variant
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(err) => write!(f, "I/O error: {err}"),
            AppError::Parse(err) => write!(f, "parse error: {err}"),
            AppError::Config(msg) => write!(f, "config error: {msg}"),
        }
    }
}

// Implement Error — wiring up source() for each variant
impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::Io(err) => Some(err),
            AppError::Parse(err) => Some(err),
            AppError::Config(_) => None,
        }
    }
}

// Implement From<io::Error> so ? works
impl From<io::Error> for AppError {
    fn from(err: io::Error) -> Self {
        AppError::Io(err)
    }
}

// Implement From<ParseIntError> so ? works
impl From<ParseIntError> for AppError {
    fn from(err: ParseIntError) -> Self {
        AppError::Parse(err)
    }
}
```

That's **~45 lines** of pure mechanical boilerplate for just three error variants. Every time you add a variant, you touch _four_ places: the enum, `Display`, `Error::source`, and a new `From` impl. This pattern is:

- **Tedious** — you're writing the same structure over and over
- **Error-prone** — forget a match arm and you get a compile error; wire up `source()` wrong and you silently lose the error chain
- **Noisy** — the boilerplate drowns out the actual error definitions

Two crates by **David Tolnay** solve this problem completely:

| Crate | Purpose | Target audience |
|-------|---------|-----------------|
| `thiserror` | Derive macros for custom error types | Library authors |
| `anyhow` | Flexible, type-erased error handling | Application authors |

Let's see how each one works.

---

## thiserror — For Library Authors

Add it to your project:

```bash
cargo add thiserror
```

Now rewrite those ~45 lines:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("config error: {0}")]
    Config(String),
}
```

**That's it.** Ten lines. The `#[derive(Error)]` macro generates all the `Display`, `Error`, and `From` impls at compile time.

### Side-by-Side Comparison

```
BEFORE (manual)                          AFTER (thiserror)
─────────────────────                    ─────────────────────
#[derive(Debug)]                         #[derive(Debug, Error)]
enum AppError {                          enum AppError {
    Io(io::Error),                           #[error("I/O error: {0}")]
    Parse(ParseIntError),                    Io(#[from] std::io::Error),
    Config(String),
}                                            #[error("parse error: {0}")]
                                             Parse(#[from] std::num::ParseIntError),
impl fmt::Display for AppError {
    fn fmt(...) { ... 6 lines }              #[error("config error: {0}")]
}                                            Config(String),
                                         }
impl std::error::Error for AppError {
    fn source(...) { ... 7 lines }
}

impl From<io::Error> for AppError {
    fn from(...) { ... 3 lines }
}

impl From<ParseIntError> for AppError {
    fn from(...) { ... 3 lines }
}

~45 lines                                ~10 lines
```

### The Key Attributes

#### `#[error("...")]` — Display Messages

This attribute goes on each variant and defines the `Display` output:

```rust
#[derive(Debug, Error)]
enum DataError {
    // Positional fields: {0}, {1}, etc.
    #[error("invalid value at index {0}: {1}")]
    InvalidValue(usize, String),

    // Named fields: {field_name}
    #[error("column `{name}` not found in table `{table}`")]
    ColumnNotFound { name: String, table: String },

    // Call methods on fields
    #[error("expected {expected} items, got {found}")]
    WrongCount { expected: usize, found: usize },

    // Display the source error directly
    #[error("failed to read data")]
    ReadFailed(#[source] std::io::Error),
}
```

Interpolation in `#[error("...")]` follows `std::fmt` syntax — you can use `{0}`, `{field}`, `{0:?}` (Debug format), and even `{0:#x}` (hex format).

#### `#[from]` — Automatic From Implementations

The `#[from]` attribute on a field generates a `From` impl so the `?` operator works:

```rust
#[derive(Debug, Error)]
enum ServiceError {
    #[error("database error")]
    Database(#[from] DatabaseError),

    #[error("network error")]
    Network(#[from] NetworkError),

    #[error("authentication failed: {reason}")]
    Auth { reason: String },
}

// These From impls are generated automatically:
// impl From<DatabaseError> for ServiceError { ... }
// impl From<NetworkError> for ServiceError { ... }
```

Now `?` converts `DatabaseError` → `ServiceError::Database` and `NetworkError` → `ServiceError::Network` automatically.

#### `#[source]` — Error Chaining Without From

Sometimes you want to mark a field as the _source_ error (for the error chain) but **don't** want a `From` impl:

```rust
#[derive(Debug, Error)]
enum ProcessError {
    #[error("failed to spawn process `{name}`")]
    SpawnFailed {
        name: String,
        #[source]
        cause: std::io::Error,
    },
}
```

Here, `cause` feeds `Error::source()` so the error chain works, but there's no blanket `From<io::Error>` (because you need the `name` field too).

> **Rule of thumb:** Use `#[from]` when the variant has a single error field. Use `#[source]` when the variant has additional context fields.

#### `#[from]` implies `#[source]`

When you write `#[from]`, thiserror automatically treats that field as `#[source]` too. You never need both on the same field.

---

## thiserror Deep Dive

### How It Works: Procedural Macros

`thiserror` is a **procedural macro crate** (a `proc-macro`). At compile time, Rust's compiler invokes the macro, which inspects your enum definition and generates the impl blocks as if you wrote them by hand. The generated code is inserted into your compilation unit — you can even see it with `cargo expand`:

```bash
cargo install cargo-expand
cargo expand  # Shows the generated code
```

For our `AppError` enum, `cargo expand` would produce output nearly identical to the 45-line manual version we wrote earlier.

### Zero-Cost Abstraction

thiserror is a **true zero-cost abstraction**:

- **No runtime dependency** — thiserror is only needed at compile time. It generates plain Rust code and then gets out of the way.
- **No extra allocations** — the generated code is the same as hand-written code.
- **Not in your binary** — thiserror doesn't add a single byte to your compiled output. Your users don't even need to know you used it.

You can verify this by checking your `Cargo.toml`. Even though you write `thiserror = "2"`, Cargo treats proc-macro crates as compile-time-only dependencies.

### Transparent Errors

The `#[error(transparent)]` attribute creates a newtype wrapper that delegates _everything_ to the inner error:

```rust
#[derive(Debug, Error)]
#[error(transparent)]
struct MyError(#[from] anyhow::Error);
```

This is useful for:
- Creating a newtype around another error type
- Bridging between error types in different layers of your code

Both `Display` and `source()` delegate directly to the inner error.

```rust
// Transparent delegates Display and source() to the inner error
#[derive(Debug, Error)]
enum ApiError {
    #[error(transparent)]
    Internal(#[from] InternalError),

    #[error("request timed out after {0} seconds")]
    Timeout(u64),
}
```

### Backtrace Support

thiserror can capture backtraces for debugging. If you include a `std::backtrace::Backtrace` field, thiserror will wire it up:

```rust
use std::backtrace::Backtrace;

#[derive(Debug, Error)]
enum AppError {
    #[error("unexpected error")]
    Unexpected {
        #[source]
        cause: Box<dyn std::error::Error + Send + Sync>,
        backtrace: Backtrace,
    },
}
```

> **Note:** `std::backtrace::Backtrace` is stable since Rust 1.65, but backtrace _capture_ requires the `RUST_BACKTRACE=1` environment variable at runtime.

### Struct Errors (Not Just Enums)

thiserror works on structs too, not just enums:

```rust
#[derive(Debug, Error)]
#[error("connection to `{host}:{port}` failed")]
struct ConnectionError {
    host: String,
    port: u16,
    #[source]
    cause: std::io::Error,
}
```

This is handy for errors that have a single meaning but carry context.

---

## anyhow — For Application Authors

If thiserror is about _defining_ precise error types, anyhow is about _not caring_ about error types. It wraps any error into a single, type-erased `anyhow::Error`.

Add it to your project:

```bash
cargo add anyhow
```

### The Basics

```rust
use anyhow::{Result, Context};

// anyhow::Result<T> is an alias for Result<T, anyhow::Error>
fn read_config(path: &str) -> Result<Config> {
    let text = std::fs::read_to_string(path)
        .context("failed to read config file")?;

    let config: Config = serde_json::from_str(&text)
        .context("failed to parse config JSON")?;

    Ok(config)
}
```

Notice:
- No custom error enum needed
- `?` converts _any_ `std::error::Error` into `anyhow::Error` automatically
- `.context()` adds a human-readable message to the error chain

### `anyhow::Result<T>`

This is just a type alias:

```rust
// In the anyhow crate:
pub type Result<T, E = Error> = std::result::Result<T, E>;
```

So `anyhow::Result<Config>` = `Result<Config, anyhow::Error>`. You use it exactly like a normal `Result`, but with a universal error type.

### `context()` and `with_context()`

`.context()` is anyhow's killer feature. It lets you attach descriptive messages to errors as they propagate up the call stack:

```rust
use anyhow::{Result, Context};
use std::fs;

fn load_user(id: u64) -> Result<User> {
    let path = format!("/data/users/{id}.json");

    let text = fs::read_to_string(&path)
        .with_context(|| format!("failed to read user file for id {id}"))?;

    let user: User = serde_json::from_str(&text)
        .with_context(|| format!("failed to parse user {id}"))?;

    if user.is_banned {
        anyhow::bail!("user {id} is banned");
    }

    Ok(user)
}
```

**`context()` vs `with_context()`:**
- `.context("static message")` — always allocates the string, even on success
- `.with_context(|| format!("user {id}"))` — lazy; only allocates when there's an error

Use `with_context()` when the message involves formatting. Use `context()` for static strings.

### `anyhow!()` — Ad-Hoc Errors

Create errors from strings on the fly:

```rust
use anyhow::anyhow;

fn validate_port(port: u16) -> Result<u16, anyhow::Error> {
    if port == 0 {
        return Err(anyhow!("port cannot be zero"));
    }
    if port < 1024 {
        return Err(anyhow!("port {port} is reserved (must be >= 1024)"));
    }
    Ok(port)
}
```

`anyhow!()` supports the same format string syntax as `format!()`.

### `bail!()` — Early Return Shorthand

`bail!()` is syntactic sugar for `return Err(anyhow!(...))`:

```rust
use anyhow::{bail, Result};

fn parse_mode(input: &str) -> Result<Mode> {
    match input {
        "fast" => Ok(Mode::Fast),
        "slow" => Ok(Mode::Slow),
        "balanced" => Ok(Mode::Balanced),
        other => bail!("unknown mode: `{other}` (expected fast, slow, or balanced)"),
    }
}
```

This is cleaner than writing `return Err(anyhow!("..."))` and reads naturally.

### `ensure!()` — Assertion That Returns Err

`ensure!()` is like `assert!()`, but returns `Err` instead of panicking:

```rust
use anyhow::{ensure, Result};

fn divide(a: f64, b: f64) -> Result<f64> {
    ensure!(b != 0.0, "cannot divide {a} by zero");
    ensure!(a.is_finite(), "dividend must be finite, got {a}");
    ensure!(b.is_finite(), "divisor must be finite, got {b}");
    Ok(a / b)
}
```

This is perfect for precondition checks — clean, readable, and non-panicking.

### Full Example: A CLI Tool

Here's a realistic example combining all anyhow features:

```rust
use anyhow::{bail, ensure, Context, Result};
use std::fs;
use std::path::Path;

#[derive(serde::Deserialize)]
struct AppConfig {
    database_url: String,
    port: u16,
    max_connections: u32,
}

fn load_config(path: &Path) -> Result<AppConfig> {
    ensure!(path.exists(), "config file not found: {}", path.display());
    ensure!(
        path.extension().map_or(false, |ext| ext == "json"),
        "config file must be JSON, got: {}",
        path.display()
    );

    let text = fs::read_to_string(path)
        .with_context(|| format!("failed to read {}", path.display()))?;

    let config: AppConfig = serde_json::from_str(&text)
        .with_context(|| format!("failed to parse {}", path.display()))?;

    // Validate the config
    ensure!(config.port >= 1024, "port {} is reserved", config.port);
    ensure!(
        config.max_connections > 0,
        "max_connections must be positive"
    );

    if config.database_url.is_empty() {
        bail!("database_url cannot be empty");
    }

    Ok(config)
}

fn connect_database(url: &str) -> Result<DatabaseConn> {
    let conn = DatabaseConn::connect(url)
        .with_context(|| format!("failed to connect to database at {url}"))?;
    Ok(conn)
}

fn main() -> Result<()> {
    let config = load_config(Path::new("config.json"))
        .context("failed to initialize application")?;

    let db = connect_database(&config.database_url)
        .context("database setup failed")?;

    println!("Server starting on port {}", config.port);
    Ok(())
}
```

If the config file contains invalid JSON, the error output looks like:

```
Error: failed to initialize application

Caused by:
    0: failed to parse config.json
    1: expected `,` or `}` at line 3 column 5
```

Every layer of context is preserved, giving the user (and the developer) a clear trail from symptom to root cause.

---

## anyhow Deep Dive

### How `anyhow::Error` Works Internally

Under the hood, `anyhow::Error` is essentially:

```rust
// Simplified conceptual model:
struct Error {
    inner: Box<ErrorImpl>,
}

struct ErrorImpl {
    error: Box<dyn std::error::Error + Send + Sync>,
    backtrace: Option<Backtrace>,
    context: Vec<Box<dyn Display + Send + Sync>>,
}
```

It's a **type-erased** container. Any error that implements `std::error::Error + Send + Sync + 'static` can be stored inside. The original type information is preserved for downcasting, but the static type is hidden behind a trait object.

Key properties:
- **Pointer-sized** — `anyhow::Error` is one pointer wide (8 bytes on 64-bit)
- **Heap-allocated** — the actual error data lives on the heap
- **Backtrace-aware** — captures a backtrace if `RUST_BACKTRACE=1` is set
- **Send + Sync** — safe to pass between threads

### Downcasting: Recovering the Original Type

Even though anyhow erases the type, you can recover it:

```rust
use anyhow::Result;
use std::io;

fn do_work() -> Result<()> {
    let _file = std::fs::File::open("missing.txt")?;
    Ok(())
}

fn main() {
    if let Err(err) = do_work() {
        // Try to downcast to a specific error type
        if let Some(io_err) = err.downcast_ref::<io::Error>() {
            match io_err.kind() {
                io::ErrorKind::NotFound => {
                    eprintln!("File not found — creating a default one...");
                }
                io::ErrorKind::PermissionDenied => {
                    eprintln!("Permission denied — run with sudo?");
                }
                _ => eprintln!("I/O error: {io_err}"),
            }
        } else {
            eprintln!("Unexpected error: {err}");
        }
    }
}
```

Downcasting methods:
- `err.downcast_ref::<T>()` → `Option<&T>` — borrow the inner error
- `err.downcast::<T>()` → `Result<T, Self>` — consume and extract the inner error
- `err.is::<T>()` → `bool` — check the type without extracting

### Error Chains

anyhow preserves the full error chain (via `.context()` and `Error::source()`). You can iterate over it:

```rust
fn print_error_chain(err: &anyhow::Error) {
    eprintln!("Error: {err}");
    for (i, cause) in err.chain().skip(1).enumerate() {
        eprintln!("  Caused by [{i}]: {cause}");
    }
}
```

### Display Formats

anyhow gives you three formatting modes:

```rust
let err: anyhow::Error = /* ... */;

// {} — top-level message only
println!("{err}");
// Output: "failed to initialize application"

// {:?} — full chain (Debug format, great for logs)
println!("{err:?}");
// Output:
// failed to initialize application
//
// Caused by:
//     0: failed to parse config.json
//     1: expected `,` or `}` at line 3 column 5
//
// Stack backtrace: ...

// {:#} — alternate format, full chain in one line
println!("{err:#}");
// Output: "failed to initialize application: failed to parse config.json:
//          expected `,` or `}` at line 3 column 5"
```

For `main() -> Result<()>`, Rust uses the `Debug` format (`{:?}`), which shows the complete chain — exactly what you want for CLI tools.

---

## thiserror vs anyhow — When to Use Which

### Decision Framework

```
                    Are you writing a library      Are you writing an
                    that other code depends on?     application (CLI, server)?
                             │                              │
                             ▼                              ▼
                    ┌─────────────────┐          ┌──────────────────┐
                    │    thiserror    │          │      anyhow      │
                    │                 │          │                  │
                    │ • Precise types │          │ • Quick & easy   │
                    │ • Callers can   │          │ • Rich context   │
                    │   match on      │          │ • No enum needed │
                    │   variants      │          │ • Great for CLI  │
                    └─────────────────┘          └──────────────────┘
```

### The Golden Rule

> **thiserror in your `lib.rs`, anyhow in your `main.rs`.**

- **Libraries** expose a public API. Your callers need to inspect errors programmatically (`match`, `downcast`, pattern matching). thiserror gives them concrete types to work with.

- **Applications** are the end of the line. Errors get reported to a human (on screen, in logs). anyhow gives you lightweight, context-rich error reporting without defining types for every failure mode.

### Comparison Table

| Feature | thiserror | anyhow |
|---------|-----------|--------|
| **Purpose** | Define custom error types | Handle errors without custom types |
| **Target** | Library authors | Application authors |
| **Error type** | Your own enum/struct | `anyhow::Error` (type-erased) |
| **Display** | `#[error("...")]` attribute | `.context()` messages |
| **From impls** | `#[from]` attribute | Automatic for all `Error` types |
| **Pattern matching** | Yes — match on variants | Via `downcast_ref::<T>()` |
| **Runtime cost** | Zero — compile-time only | Small — heap allocation |
| **Error chains** | Via `#[source]` | Via `.context()` |
| **Backtrace** | `#[backtrace]` attribute | Automatic capture |
| **Ad-hoc errors** | No | `anyhow!()`, `bail!()`, `ensure!()` |
| **Dependency** | Compile-time only | Runtime dependency |

### Using Both Together

You can — and often should — use both in the same project:

```rust
// In src/lib.rs — thiserror for the public API
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DatabaseError {
    #[error("connection failed: {0}")]
    Connection(#[source] std::io::Error),

    #[error("query failed: {query}")]
    Query { query: String, #[source] cause: SqlError },

    #[error("record not found: {0}")]
    NotFound(String),
}

pub fn get_user(id: u64) -> Result<User, DatabaseError> {
    // ...
}
```

```rust
// In src/main.rs — anyhow for the application
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let user = mylib::get_user(42)
        .context("failed to load current user")?;

    println!("Welcome, {}!", user.name);
    Ok(())
}
```

The library exposes precise error types. The application wraps them in anyhow for convenient reporting. Everyone wins.

### Quick Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│ "Should I define an error type here?"                        │
│                                                              │
│   YES → use thiserror                                        │
│     • Public library API                                     │
│     • Callers match on specific errors                       │
│     • You want structured, documented error variants         │
│                                                              │
│   NO → use anyhow                                            │
│     • CLI/server application code                            │
│     • Errors are reported to a human, not matched on         │
│     • You want .context("what happened") and move on         │
└──────────────────────────────────────────────────────────────┘
```

---

## Other Error Crates

While thiserror and anyhow are the most widely used, the Rust ecosystem has other notable crates for specific needs.

### `eyre` — Customizable Error Reports

[eyre](https://crates.io/crates/eyre) is a fork of anyhow that lets you customize how errors are displayed. On its own it's similar to anyhow, but it shines when paired with **`color-eyre`**:

```rust
use color_eyre::eyre::{Result, WrapErr};

fn main() -> Result<()> {
    color_eyre::install()?;  // Install the fancy error handler

    let config = load_config("app.toml")
        .wrap_err("could not initialize application")?;

    Ok(())
}
```

When an error occurs, `color-eyre` produces beautifully formatted, colorized output with:
- Syntax-highlighted error chains
- Span traces (with `tracing` integration)
- Suggestions and help text
- Custom sections

**When to use:** You want richer, prettier error reports than anyhow provides — especially in CLI tools where the user experience matters.

### `miette` — Diagnostic Errors with Source Snippets

[miette](https://crates.io/crates/miette) takes error reporting even further by showing **source code snippets** alongside errors, similar to how the Rust compiler displays errors:

```rust
use miette::{Diagnostic, SourceSpan};
use thiserror::Error;

#[derive(Debug, Error, Diagnostic)]
#[error("invalid syntax")]
#[diagnostic(code(parser::syntax_error), help("expected a closing bracket"))]
struct SyntaxError {
    #[source_code]
    src: String,

    #[label("this bracket is never closed")]
    span: SourceSpan,
}
```

This produces output like:

```
  × invalid syntax
   ╭─[input.txt:3:8]
 3 │ fn main() {
   ·          ┬
   ·          ╰── this bracket is never closed
   ╰────
  help: expected a closing bracket
```

**When to use:** You're building a compiler, linter, config parser, or any tool where pointing to a specific location in source text helps the user fix the problem.

### `snafu` — Context Selectors

[snafu](https://crates.io/crates/snafu) is an alternative to thiserror that takes a different approach. Instead of `#[from]`, it uses **context selectors** — dedicated types for adding context:

```rust
use snafu::{ResultExt, Snafu};

#[derive(Debug, Snafu)]
enum Error {
    #[snafu(display("failed to read config from {path}"))]
    ReadConfig {
        path: String,
        source: std::io::Error,
    },
}

fn load() -> Result<String, Error> {
    let path = "config.toml";
    std::fs::read_to_string(path)
        .context(ReadConfigSnafu { path })?  // Context selector
    // ReadConfigSnafu is auto-generated
}
```

**When to use:** You prefer the context selector pattern over `#[from]`, or you want more control over how context is attached to errors. Some large projects (like the InfluxDB IOx database) use snafu.

### Comparison at a Glance

| Crate | Like | Best for |
|-------|------|----------|
| `thiserror` | — | Library error types |
| `anyhow` | — | Application error handling |
| `eyre` / `color-eyre` | anyhow + pretty | CLI tools with rich output |
| `miette` | anyhow + source spans | Compilers, linters, parsers |
| `snafu` | thiserror + context selectors | Teams that prefer the selector pattern |

---

## The Rust Error Handling Ecosystem Evolution

Understanding the history helps you appreciate why thiserror and anyhow exist and why they became the standard.

### Timeline

```
2015 ─── Rust 1.0 stable
         • Raw Result<T, E> with manual impl Display, Error, From
         • Extremely verbose — community immediately sought solutions

2016 ─── error-chain
         • Macro-based, generated error types with chaining
         • Very popular for a time, but macro-heavy and opaque
         • Hard to understand what was generated

2017 ─── failure (by withoutboats)
         • Introduced failure::Error — a type-erased error (like anyhow)
         • Also had #[derive(Fail)] — like thiserror
         • Tried to do everything in one crate
         • Issues: split ecosystem (failure::Fail vs std::error::Error),
           heavy dependency, compatibility concerns

2019 ─── thiserror + anyhow (by David Tolnay, dtolnay)
         • Split the problem in two: defining errors vs handling errors
         • Built on std::error::Error — no competing trait
         • Minimal, focused, zero-cost (thiserror), lightweight (anyhow)
         • Immediately adopted by the community

2020 ─── eyre (fork of anyhow by Jane Lusby / yaahc)
         • Added customizable error reporters
         • color-eyre became popular for CLIs

2021 ─── miette (by Kat Marchán)
         • Diagnostic-style errors with source snippets
         • Popular for dev tools

2024+ ── thiserror 2.0
         • Support for generic error types, better diagnostics
         • Remains the community standard

Today ── thiserror + anyhow dominate
         • error-chain: deprecated
         • failure: deprecated
         • thiserror: ~250M downloads
         • anyhow: ~200M downloads
```

### Why thiserror/anyhow Won

1. **Simplicity** — Each crate does one thing well. thiserror generates impls. anyhow erases types. No trying to be everything to everyone.

2. **Zero-cost** — thiserror is compile-time only. anyhow is minimal at runtime. Neither imposes a "framework" on your code.

3. **Standards-compatible** — Both work with `std::error::Error`. No competing traits, no ecosystem split.

4. **Maintenance** — David Tolnay is one of the most prolific and reliable maintainers in the Rust ecosystem (also maintains `serde`, `syn`, `quote`, `proc-macro2`, and dozens more).

5. **Complementary** — They solve different problems for different audiences but work together seamlessly. The "thiserror in libs, anyhow in apps" mantra is easy to remember and apply.

---

## Exercises

### Exercise 1: Rewrite with thiserror

Take this manual error type and rewrite it using thiserror. The behavior should be identical.

```rust
use std::fmt;
use std::io;
use std::num::ParseFloatError;

#[derive(Debug)]
enum MathError {
    InvalidInput(String),
    DivisionByZero,
    Io(io::Error),
    ParseFloat(ParseFloatError),
}

impl fmt::Display for MathError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MathError::InvalidInput(msg) => write!(f, "invalid input: {msg}"),
            MathError::DivisionByZero => write!(f, "division by zero"),
            MathError::Io(err) => write!(f, "I/O error: {err}"),
            MathError::ParseFloat(err) => write!(f, "parse error: {err}"),
        }
    }
}

impl std::error::Error for MathError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            MathError::Io(err) => Some(err),
            MathError::ParseFloat(err) => Some(err),
            _ => None,
        }
    }
}

impl From<io::Error> for MathError {
    fn from(err: io::Error) -> Self { MathError::Io(err) }
}

impl From<ParseFloatError> for MathError {
    fn from(err: ParseFloatError) -> Self { MathError::ParseFloat(err) }
}
```

<details>
<summary>Solution</summary>

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum MathError {
    #[error("invalid input: {0}")]
    InvalidInput(String),

    #[error("division by zero")]
    DivisionByZero,

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error: {0}")]
    ParseFloat(#[from] std::num::ParseFloatError),
}
```

~45 lines → ~12 lines, identical behavior.

</details>

### Exercise 2: CLI with anyhow

Write a small program using anyhow that:
1. Reads a file path from the command line arguments
2. Reads the file contents
3. Parses each line as an `i64`
4. Prints the sum

Use `.context()` or `.with_context()` on every fallible operation. Use `ensure!()` to check that at least one argument was provided. Use `bail!()` if the file is empty.

<details>
<summary>Solution</summary>

```rust
use anyhow::{bail, ensure, Context, Result};
use std::env;
use std::fs;

fn run() -> Result<()> {
    let args: Vec<String> = env::args().collect();
    ensure!(args.len() >= 2, "usage: {} <file>", args[0]);

    let path = &args[1];
    let text = fs::read_to_string(path)
        .with_context(|| format!("failed to read file `{path}`"))?;

    if text.trim().is_empty() {
        bail!("file `{path}` is empty — nothing to sum");
    }

    let mut sum: i64 = 0;
    for (i, line) in text.lines().enumerate() {
        let n: i64 = line.trim().parse()
            .with_context(|| format!("line {} is not a valid integer: `{line}`", i + 1))?;
        sum += n;
    }

    println!("Sum: {sum}");
    Ok(())
}

fn main() -> Result<()> {
    run()
}
```

If line 3 contains "abc", the error looks like:
```
Error: line 3 is not a valid integer: `abc`

Caused by:
    invalid digit found in string
```

</details>

### Exercise 3: Library + Application

Build a two-layer design:

1. Define a `WeatherError` enum using thiserror with variants:
   - `NetworkError` (wraps `reqwest::Error` or `io::Error`)
   - `ParseError` (wraps `serde_json::Error`)
   - `ApiError { status: u16, message: String }`

2. Write a `get_weather(city: &str) -> Result<Weather, WeatherError>` function (the body can be stubbed with `todo!()`)

3. Write a `main()` function that calls `get_weather` and uses anyhow to wrap the result with `.context()`

<details>
<summary>Solution</summary>

```rust
// ---- Library layer (thiserror) ----
use thiserror::Error;

#[derive(Debug, Error)]
pub enum WeatherError {
    #[error("network error")]
    Network(#[from] std::io::Error),

    #[error("failed to parse weather response")]
    Parse(#[from] serde_json::Error),

    #[error("API error (status {status}): {message}")]
    Api { status: u16, message: String },
}

pub struct Weather {
    pub city: String,
    pub temp_celsius: f64,
}

pub fn get_weather(city: &str) -> Result<Weather, WeatherError> {
    // In real code, this would make an HTTP request.
    // For the exercise, simulate an API error:
    Err(WeatherError::Api {
        status: 404,
        message: format!("city `{city}` not found"),
    })
}

// ---- Application layer (anyhow) ----
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let city = "Atlantis";

    let weather = get_weather(city)
        .with_context(|| format!("failed to fetch weather for `{city}`"))?;

    println!("{}: {:.1}°C", weather.city, weather.temp_celsius);
    Ok(())
}
```

Output:
```
Error: failed to fetch weather for `Atlantis`

Caused by:
    API error (status 404): city `Atlantis` not found
```

thiserror defines the precise error. anyhow wraps it with human context.

</details>

---

## Summary

| Concept | Key takeaway |
|---------|-------------|
| **The boilerplate problem** | Manual `Display` + `Error` + `From` impls are tedious and error-prone |
| **thiserror** | Derive macro that generates all the boilerplate at compile time — zero-cost |
| **`#[error("...")]`** | Defines the `Display` message with field interpolation |
| **`#[from]`** | Generates `From` impl (and marks as `source`) — enables `?` |
| **`#[source]`** | Marks a field as the error source without generating `From` |
| **`#[error(transparent)]`** | Delegates Display and source to the inner error |
| **anyhow** | Type-erased error handling with rich context — one error type for everything |
| **`context()` / `with_context()`** | Attach human-readable messages to the error chain |
| **`anyhow!()` / `bail!()` / `ensure!()`** | Create ad-hoc errors, early returns, and precondition checks |
| **Downcasting** | Recover the original error type from `anyhow::Error` with `downcast_ref` |
| **The golden rule** | thiserror in `lib.rs`, anyhow in `main.rs` |
| **Other crates** | `eyre`/`color-eyre` for pretty output, `miette` for diagnostics, `snafu` for context selectors |
| **Ecosystem history** | error-chain → failure → thiserror + anyhow (the winner) |

> **The mantra:** Define errors with **thiserror**. Handle errors with **anyhow**. Use both when it makes sense.

---

**Previous:** [← Custom Error Types](./05-custom-error-types.md) · **Next:** [Best Practices →](./07-best-practices.md)

<p align="center"><i>Tutorial 6 of 7 — Stage 7: Error Handling</i></p>

# The `?` Operator — Error Propagation Made Beautiful 🎯

> **"The `?` operator transforms error handling from a chore into an art — explicit, concise, and elegant all at once."**

---

## Table of Contents

- [The Problem: Verbose Error Handling](#the-problem-verbose-error-handling)
- [Enter the ? Operator](#enter-the--operator)
- [How ? Works Under the Hood](#how--works-under-the-hood)
- [Before and After ? — Side by Side](#before-and-after----side-by-side)
- [Chaining ? Operations](#chaining--operations)
- [The From Trait — Automatic Error Conversion](#the-from-trait--automatic-error-conversion)
- [? with Option](#-with-option)
- [? in main()](#-in-main)
- [The History of ?](#the-history-of-)
- [Error Propagation in Other Languages](#error-propagation-in-other-languages)
- [Common Patterns with ?](#common-patterns-with-)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Problem: Verbose Error Handling

Before we meet the `?` operator, let's feel the pain it was designed to cure. Imagine you're
writing a function that reads a username from a file, trims it, and returns it. With raw `match`
statements, you'd write something like this:

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let file_result = File::open("username.txt");

    let mut file = match file_result {
        Ok(f) => f,
        Err(e) => return Err(e),
    };

    let mut username = String::new();

    let read_result = file.read_to_string(&mut username);

    match read_result {
        Ok(_) => Ok(username),
        Err(e) => Err(e),
    }
}
```

Two `match` expressions for two operations. It works, but it's noisy. The important business
logic — "open a file, read its content, return the string" — is buried under boilerplate.

Now imagine a function that has five fallible steps:

```rust
fn load_config() -> Result<Config, io::Error> {
    let path_result = find_config_path();
    let path = match path_result {
        Ok(p) => p,
        Err(e) => return Err(e),
    };

    let file_result = File::open(&path);
    let mut file = match file_result {
        Ok(f) => f,
        Err(e) => return Err(e),
    };

    let mut contents = String::new();
    let read_result = file.read_to_string(&mut contents);
    match read_result {
        Ok(_) => {},
        Err(e) => return Err(e),
    };

    let parsed_result = parse_config(&contents);
    let config = match parsed_result {
        Ok(c) => c,
        Err(e) => return Err(e),
    };

    let validated_result = validate_config(config);
    match validated_result {
        Ok(c) => Ok(c),
        Err(e) => Err(e),
    }
}
```

Five operations, five match blocks, an ocean of `Err(e) => return Err(e)`. Every match block
says the exact same thing: *"If this failed, just pass the error along."* That's a pattern.
And in Rust, patterns become operators.

---

## Enter the `?` Operator

The `?` operator is a postfix operator that you place after an expression that returns
`Result<T, E>` (or `Option<T>`). It does exactly two things:

- **If `Ok(val)`** → unwrap the value and continue execution.
- **If `Err(e)`** → return **immediately** from the enclosing function with that error.

Here it is in action:

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut file = File::open("username.txt")?;
    let mut username = String::new();
    file.read_to_string(&mut username)?;
    Ok(username)
}
```

That's it. Three lines of real logic. No match blocks, no nested indentation. The `?` at the
end of `File::open("username.txt")?` says: *"If this is an error, return it. Otherwise, give me
the file handle."*

### The Rules

1. `?` can only be used in functions that return `Result` (or `Option` — more on that later).
2. The error type of the expression must be convertible into the error type of the function's return.
3. `?` is an expression — it evaluates to the `Ok` value.

```rust
// This WON'T compile — main returns () by default, not Result
fn main() {
    let file = File::open("hello.txt")?; // ERROR!
}
```

```
error[E0277]: the `?` operator can only be used in a function
that returns `Result` or `Option`
```

We'll cover how to use `?` in `main()` shortly.

---

## How `?` Works Under the Hood

The `?` operator is syntactic sugar. When you write:

```rust
let val = some_expression?;
```

The compiler transforms it into roughly:

```rust
let val = match some_expression {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
};
```

Notice `From::from(e)` — not just `Err(e)`. This is the secret sauce. The `?` operator
calls the `From` trait to **automatically convert** the error type of the inner expression
into the error type of the enclosing function's return. This means you can use `?` across
different error types, as long as conversions exist.

Let's be precise about the desugaring:

```rust
// What you write:
let file = File::open("data.txt")?;

// What the compiler generates (simplified):
let file = match File::open("data.txt") {
    Ok(val) => val,
    Err(err) => {
        // Convert the error type using the From trait
        let converted: ReturnErrorType = From::from(err);
        return Err(converted);
    }
};
```

The full desugaring actually goes through the `Try` trait (stabilized in nightly as of writing),
but for `Result`, the `From::from` mental model is accurate and practical.

### Key Takeaway

The `?` operator is **not** just a shortcut for pattern matching. It performs **error type
conversion** via `From`. This is what makes it composable across libraries and error types.

---

## Before and After `?` — Side by Side

Let's see that five-step config function rewritten with `?`:

### Before (Nested Matches)

```rust
fn load_config() -> Result<Config, io::Error> {
    let path_result = find_config_path();
    let path = match path_result {
        Ok(p) => p,
        Err(e) => return Err(e),
    };

    let file_result = File::open(&path);
    let mut file = match file_result {
        Ok(f) => f,
        Err(e) => return Err(e),
    };

    let mut contents = String::new();
    let read_result = file.read_to_string(&mut contents);
    match read_result {
        Ok(_) => {},
        Err(e) => return Err(e),
    };

    let parsed_result = parse_config(&contents);
    let config = match parsed_result {
        Ok(c) => c,
        Err(e) => return Err(e),
    };

    let validated_result = validate_config(config);
    match validated_result {
        Ok(c) => Ok(c),
        Err(e) => Err(e),
    }
}
```

**Lines: ~30** | **Match blocks: 5** | **Noise level: 🔴 High**

### After (With `?`)

```rust
fn load_config() -> Result<Config, io::Error> {
    let path = find_config_path()?;
    let mut file = File::open(&path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    let config = parse_config(&contents)?;
    validate_config(config)
}
```

**Lines: ~7** | **Match blocks: 0** | **Noise level: 🟢 Minimal**

Same logic. Same safety. Same compile-time guarantees. The `?` version reads like a
**description of what the function does**, not how it handles every failure case. The
error propagation is still explicit — you can *see* every `?` — but it doesn't dominate
the code.

---

## Chaining `?` Operations

Because `?` is an expression that evaluates to the unwrapped value, you can chain it
with method calls.

### Chain on the Same Line

```rust
use std::fs;

fn read_username() -> Result<String, io::Error> {
    // fs::read_to_string does open + read in one step
    let username = fs::read_to_string("username.txt")?;
    Ok(username)
}
```

Or even shorter:

```rust
fn read_username() -> Result<String, io::Error> {
    fs::read_to_string("username.txt")
}
```

Wait — no `?` needed here! Since the return type already matches, we just return the
`Result` directly. Use `?` when you need to **do something** with the unwrapped value.

### Chain Multiple Methods

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username() -> Result<String, io::Error> {
    let mut buf = String::new();
    File::open("username.txt")?.read_to_string(&mut buf)?;
    Ok(buf)
}
```

Here, `File::open("username.txt")?` either returns a `File` or bails out. If it succeeds,
`.read_to_string(&mut buf)?` is immediately called on that file handle. Two fallible
operations, one line, two `?` operators.

### Longer Chains

```rust
fn first_line_length(path: &str) -> Result<usize, io::Error> {
    let content = fs::read_to_string(path)?;
    let first_line = content.lines().next().ok_or(io::Error::new(
        io::ErrorKind::InvalidData,
        "file is empty",
    ))?;
    Ok(first_line.len())
}
```

Notice the `.ok_or(...)?` pattern — converting an `Option` to a `Result` so we can use `?`
on it. This is extremely common in real-world Rust.

### A Word of Caution

Don't chain **too** aggressively. If a line has three or more `?` operators, consider
splitting it for readability:

```rust
// Arguably too dense:
let val = get_connection()?.query("SELECT ...")?.first()?.parse()?;

// More readable:
let conn = get_connection()?;
let rows = conn.query("SELECT ...")?;
let first = rows.first()?;
let val = first.parse()?;
```

Clarity always wins over cleverness.

---

## The `From` Trait — Automatic Error Conversion

This is the mechanism that makes `?` truly powerful. Let's understand it fully.

### The Problem

Suppose your function returns `Result<T, AppError>`, but inside it you call a function
that returns `Result<_, io::Error>` and another that returns `Result<_, serde_json::Error>`.
Without any conversion, `?` wouldn't compile — the error types don't match.

### The Solution: `From` Implementations

```rust
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
}

// Tell Rust how to convert io::Error → AppError
impl From<io::Error> for AppError {
    fn from(err: io::Error) -> AppError {
        AppError::Io(err)
    }
}

// Tell Rust how to convert ParseIntError → AppError
impl From<ParseIntError> for AppError {
    fn from(err: ParseIntError) -> AppError {
        AppError::Parse(err)
    }
}
```

Now `?` works seamlessly across both error types:

```rust
use std::fs;

fn read_and_parse_number(path: &str) -> Result<i64, AppError> {
    let content = fs::read_to_string(path)?;       // io::Error → AppError via From
    let number = content.trim().parse::<i64>()?;    // ParseIntError → AppError via From
    Ok(number)
}
```

Both `?` operators compile because `From` conversions exist. The compiler inserts
`From::from(e)` automatically — you never write it yourself.

### The Conversion Flow

```
File::open("data.txt")?
         │
         ├── Ok(file) → unwrap, continue
         │
         └── Err(io::Error) 
                  │
                  ▼
          From::from(io::Error)
                  │
                  ▼
            AppError::Io(...)
                  │
                  ▼
          return Err(AppError::Io(...))
```

### The `Box<dyn Error>` Shortcut

If you don't want to define a custom error type, `Box<dyn std::error::Error>` serves as
a universal error type. The standard library provides `From` implementations for converting
any `Error` type into `Box<dyn Error>`:

```rust
use std::error::Error;
use std::fs;

fn do_everything() -> Result<i64, Box<dyn Error>> {
    let content = fs::read_to_string("number.txt")?;   // io::Error → Box<dyn Error>
    let n: i64 = content.trim().parse()?;               // ParseIntError → Box<dyn Error>
    Ok(n * 2)
}
```

No custom error type, no `From` implementations. The trade-off: you lose the ability to
match on specific error variants without downcasting.

### Quick Reference: When Does `?` Compile?

| Expression Type | Function Returns | Compiles? | Why |
|---|---|---|---|
| `Result<T, io::Error>` | `Result<_, io::Error>` | ✅ | Same type |
| `Result<T, io::Error>` | `Result<_, AppError>` | ✅ | If `From<io::Error> for AppError` exists |
| `Result<T, io::Error>` | `Result<_, Box<dyn Error>>` | ✅ | Blanket `From` impl |
| `Result<T, io::Error>` | `Result<_, String>` | ❌ | No `From<io::Error> for String` |
| `Result<T, io::Error>` | `Option<T>` | ❌ | Incompatible carrier types |

---

## `?` with `Option`

Since Rust 1.22, the `?` operator also works with `Option<T>`:

- **If `Some(val)`** → unwrap the value and continue.
- **If `None`** → return `None` immediately from the function.

```rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    let first_line = text.lines().next()?;
    first_line.chars().last()
}
```

Here, `text.lines().next()` returns `Option<&str>`. The `?` unwraps the `Some` case or
returns `None` early.

### The Rules for `?` with `Option`

1. The enclosing function must return `Option<T>`.
2. `?` on `Option` **cannot** be used in a function that returns `Result` (and vice versa)
   unless you convert first.

```rust
// ERROR: can't use ? on Option in a function returning Result
fn bad_mix() -> Result<char, String> {
    let c = "hello".chars().next()?;  // Won't compile!
    Ok(c)
}
```

### Converting Between `Option` and `Result`

Use `.ok_or()` or `.ok_or_else()` to convert `Option` → `Result`:

```rust
fn first_char(text: &str) -> Result<char, String> {
    let c = text.chars().next().ok_or("empty string".to_string())?;
    Ok(c)
}
```

Use `.ok()` to convert `Result` → `Option` (discarding the error):

```rust
fn try_parse(s: &str) -> Option<i64> {
    let n = s.trim().parse::<i64>().ok()?;
    Some(n)
}
```

### Chaining `?` with `Option`

```rust
fn get_middle_name(full_name: &str) -> Option<&str> {
    let mut parts = full_name.split_whitespace();
    let _first = parts.next()?;
    let middle = parts.next()?;  // None if only one name
    // Verify there's a last name too (at least 3 parts)
    let _last = parts.next()?;
    Some(middle)
}

fn main() {
    assert_eq!(get_middle_name("John Michael Smith"), Some("Michael"));
    assert_eq!(get_middle_name("John Smith"), None);
    assert_eq!(get_middle_name("Cher"), None);
}
```

---

## `?` in `main()`

By default, `main()` returns `()`, so you can't use `?` directly. But since Rust 1.26,
`main()` can return a `Result`:

```rust
use std::error::Error;
use std::fs;

fn main() -> Result<(), Box<dyn Error>> {
    let content = fs::read_to_string("config.txt")?;
    let port: u16 = content.trim().parse()?;
    println!("Starting server on port {port}");
    Ok(())
}
```

If `main()` returns `Err`, Rust prints the error's `Debug` representation and exits with
a non-zero status code:

```
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

### Using a Custom Error Type in `main()`

Any type that implements `std::process::Termination` can be returned from `main()`. For
`Result`, both `Ok` and `Err` types must satisfy certain constraints. In practice,
`Result<(), Box<dyn Error>>` is the most common pattern:

```rust
use std::error::Error;

type Result<T> = std::result::Result<T, Box<dyn Error>>;

fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().collect();
    let filename = args.get(1).ok_or("Usage: program <filename>")?;
    let content = std::fs::read_to_string(filename)?;
    println!("{content}");
    Ok(())
}
```

### `ExitCode` Alternative

For more control over the exit code, you can use `std::process::ExitCode`:

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("Error: {e}");
            ExitCode::FAILURE
        }
    }
}

fn run() -> Result<(), Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string("data.txt")?;
    println!("Read {} bytes", content.len());
    Ok(())
}
```

This is a very common pattern in production Rust: keep `main()` thin, delegate to a `run()`
function that returns `Result`, and handle the error display in `main()`.

---

## The History of `?`

The `?` operator didn't appear overnight. It's the result of years of language evolution.

### The `try!` Macro Era (Rust 1.0 — 2015)

Rust 1.0 shipped with the `try!()` macro, which did exactly what `?` does today:

```rust
// Rust 1.0 style (2015)
use std::fs::File;
use std::io::{self, Read};

fn read_username() -> Result<String, io::Error> {
    let mut file = try!(File::open("username.txt"));
    let mut username = String::new();
    try!(file.read_to_string(&mut username));
    Ok(username)
}
```

`try!` was a macro that expanded to the same match + return pattern:

```rust
macro_rules! try {
    ($expr:expr) => {
        match $expr {
            Ok(val) => val,
            Err(err) => return Err(From::from(err)),
        }
    };
}
```

### The `?` Operator Arrives (Rust 1.13 — November 2016)

RFC 243 proposed the `?` operator as a language-level replacement for `try!`. It was
stabilized in Rust 1.13, released November 10, 2016.

```rust
// Rust 1.13+ style (2016)
fn read_username() -> Result<String, io::Error> {
    let mut file = File::open("username.txt")?;
    let mut username = String::new();
    file.read_to_string(&mut username)?;
    Ok(username)
}
```

### `try!` vs `?` Comparison

```rust
// try! — macro, prefix, can't chain
let file = try!(File::open("data.txt"));

// ? — operator, postfix, chainable
let file = File::open("data.txt")?;
```

| Feature | `try!` | `?` |
|---|---|---|
| Syntax | `try!(expr)` | `expr?` |
| Type | Macro | Operator |
| Position | Prefix (wraps expression) | Postfix (follows expression) |
| Chainable | ❌ `try!(try!(File::open(p)).read(...))` | ✅ `File::open(p)?.read(...)?` |
| Works with Option | ❌ | ✅ (since 1.22) |
| Deprecated | ✅ (edition 2018) | ❌ — the standard |
| Available since | 1.0 (2015) | 1.13 (Nov 2016) |

The `try!` macro was soft-deprecated in the 2018 edition of Rust. In modern Rust, always
use `?`.

### Why `?` Won

1. **Postfix syntax** — you read left to right: `get_thing()?.do_something()?`. With `try!`,
   you'd write `try!(try!(get_thing()).do_something())` — inside-out reading.
2. **Chainable** — method chains are natural with postfix operators.
3. **Not a macro** — it's a real language construct, so the compiler gives better error
   messages and optimizations.
4. **Extensible** — the underlying `Try` trait can be implemented for custom types (nightly).

### Similar Concepts in Other Languages

| Language | Mechanism | Notes |
|---|---|---|
| **Haskell** | `do` notation + monadic bind | General monadic chaining; `?` is conceptually similar for the `Either` monad |
| **Swift** | `try` keyword | `let data = try loadFile()` — similar explicit marking of fallible calls |
| **Kotlin** | `?.let { }` / `?:` | Null-safety operators for `Option`-like behavior, not errors |
| **F#** | Computation expressions | `let!` in `result { }` blocks — same idea, different syntax |
| **OCaml** | `let*` binding operator | Monadic let bindings — very close in spirit |

---

## Error Propagation in Other Languages

Let's see how different languages handle the "propagate this error to the caller" problem,
and why Rust's `?` hits a sweet spot.

### Java — Checked Exceptions

```java
// Java: must declare every exception in the signature
public String readUsername() throws IOException {
    BufferedReader reader = new BufferedReader(
        new FileReader("username.txt")
    );
    return reader.readLine();
}
```

Exceptions propagate **implicitly** through the call stack. The `throws` clause tells callers
what might go wrong, but many Java developers simply catch `Exception` or use unchecked
exceptions to avoid the verbosity. The error path is invisible at the call site.

### Go — `if err != nil`

```go
// Go: explicit but extremely repetitive
func readUsername() (string, error) {
    data, err := os.ReadFile("username.txt")
    if err != nil {
        return "", err
    }
    return strings.TrimSpace(string(data)), nil
}
```

Go forces you to check every error, which is good. But there's no shorthand — every error
check is three lines of `if err != nil { return ..., err }`. In a function with 10 fallible
calls, that's 30 lines of boilerplate.

### Python — Invisible Propagation

```python
# Python: errors propagate silently
def read_username():
    with open("username.txt") as f:
        return f.read().strip()
```

Clean and short — but there's nothing in this code that tells you it can fail. Exceptions
fly up the call stack invisibly. You only know about potential errors by reading docs or
running the code.

### Swift — `try` Keyword

```swift
// Swift: must mark fallible calls with try
func readUsername() throws -> String {
    let data = try String(contentsOfFile: "username.txt")
    return data.trimmingCharacters(in: .whitespacesAndNewlines)
}
```

Swift's approach is close to Rust's `?`. The `try` keyword marks fallible calls, and
`throws` in the signature tells callers. But Swift uses exceptions under the hood, and the
error type is erased (you get `Error`, not a specific type).

### Comparison Table

| Feature | Rust `?` | Java `throws` | Go `if err` | Python | Swift `try` |
|---|---|---|---|---|---|
| **Explicit at call site** | ✅ `?` visible | ❌ Implicit propagation | ✅ `if err != nil` | ❌ Invisible | ✅ `try` visible |
| **Concise** | ✅ One character | ✅ (implicit) | ❌ 3 lines per error | ✅ (implicit) | ✅ One keyword |
| **Type-safe errors** | ✅ Concrete types | ⚠️ Class hierarchy | ⚠️ `error` interface | ❌ Anything | ⚠️ `Error` protocol |
| **Composable** | ✅ `From` trait | ❌ Must catch & rethrow | ❌ Manual wrapping | N/A | ⚠️ Limited |
| **Zero-cost** | ✅ No runtime overhead | ❌ Stack unwinding | ✅ Value-based | ❌ Stack unwinding | ❌ Stack unwinding |
| **Chainable** | ✅ Postfix | N/A | ❌ | N/A | ❌ Prefix |

Rust's `?` is **explicit AND concise** — a combination that no other mainstream language
achieves. You can see every point where an error might occur (the `?` markers), but they
don't drown out the logic.

---

## Common Patterns with `?`

### Pattern 1: Reading and Parsing a Config File

```rust
use std::collections::HashMap;
use std::error::Error;
use std::fs;

#[derive(Debug)]
struct Config {
    settings: HashMap<String, String>,
}

fn load_config(path: &str) -> Result<Config, Box<dyn Error>> {
    let content = fs::read_to_string(path)?;
    let mut settings = HashMap::new();

    for line in content.lines() {
        let line = line.trim();

        // Skip empty lines and comments
        if line.is_empty() || line.starts_with('#') {
            continue;
        }

        let (key, value) = line
            .split_once('=')
            .ok_or_else(|| format!("Invalid config line: {line}"))?;

        settings.insert(
            key.trim().to_string(),
            value.trim().to_string(),
        );
    }

    Ok(Config { settings })
}
```

Notice how `?` is used on both `fs::read_to_string` (returns `io::Error`) and `.ok_or_else()`
(returns a `String`-based error). Both convert to `Box<dyn Error>` automatically.

### Pattern 2: Database Query and Transformation Pipeline

```rust
use std::error::Error;

struct User {
    id: u64,
    name: String,
    email: String,
}

struct UserProfile {
    display_name: String,
    email_domain: String,
}

fn get_user_profile(db: &Database, user_id: u64) -> Result<UserProfile, Box<dyn Error>> {
    let user = db.query_user(user_id)?;              // DB error
    let verified = verify_user_active(&user)?;        // Validation error
    let email_domain = extract_domain(&verified.email)?;  // Parse error

    Ok(UserProfile {
        display_name: format_display_name(&verified.name),
        email_domain,
    })
}

fn extract_domain(email: &str) -> Result<String, Box<dyn Error>> {
    let domain = email
        .split('@')
        .nth(1)
        .ok_or("Invalid email: no @ symbol")?;
    Ok(domain.to_string())
}
```

Each step can fail for different reasons (database, validation, parsing), but `?` handles
them all uniformly.

### Pattern 3: HTTP Request and JSON Parsing

```rust
use std::error::Error;
use std::collections::HashMap;

// Using a hypothetical HTTP client
fn fetch_weather(city: &str) -> Result<f64, Box<dyn Error>> {
    let url = format!("https://api.weather.example/v1/current?city={city}");
    let response = http::get(&url)?;                          // Network error

    if response.status() != 200 {
        return Err(format!("API returned status {}", response.status()).into());
    }

    let body = response.text()?;                               // IO error
    let json: HashMap<String, serde_json::Value> = 
        serde_json::from_str(&body)?;                          // JSON parse error

    let temp = json
        .get("temperature")
        .ok_or("Missing 'temperature' field")?                 // Option → Result
        .as_f64()
        .ok_or("'temperature' is not a number")?;              // Option → Result

    Ok(temp)
}
```

Five different error types, zero match blocks. Every `?` is a potential early return, and
you can see exactly where the function might bail. This is the power of `?`.

### Pattern 4: The Builder Pattern with Validation

```rust
use std::error::Error;
use std::net::IpAddr;

struct ServerConfig {
    host: IpAddr,
    port: u16,
    max_connections: usize,
}

fn build_server_config(
    host: &str,
    port: &str,
    max_conn: &str,
) -> Result<ServerConfig, Box<dyn Error>> {
    let host: IpAddr = host.parse()?;
    let port: u16 = port.parse()?;
    let max_connections: usize = max_conn.parse()?;

    if port == 0 {
        return Err("Port cannot be zero".into());
    }

    if max_connections == 0 {
        return Err("Max connections must be positive".into());
    }

    Ok(ServerConfig {
        host,
        port,
        max_connections,
    })
}
```

Three `.parse()?` calls — each can fail with a different `ParseError` variant, and all
convert to `Box<dyn Error>`. The explicit validation errors use `.into()` to convert
`&str` → `Box<dyn Error>`.

---

## Exercises

### Exercise 1: File Statistics

Write a function `file_stats` that takes a file path, reads the file, and returns a struct
with the number of lines, words, and characters. Use the `?` operator for all fallible
operations.

```rust
use std::error::Error;
use std::fs;

#[derive(Debug, PartialEq)]
struct FileStats {
    lines: usize,
    words: usize,
    chars: usize,
}

fn file_stats(path: &str) -> Result<FileStats, Box<dyn Error>> {
    // TODO: Implement this function using ?
    // 1. Read the file contents
    // 2. Count lines, words, and characters
    // 3. Return the FileStats struct
    todo!()
}

fn main() -> Result<(), Box<dyn Error>> {
    let stats = file_stats("sample.txt")?;
    println!("{stats:?}");
    Ok(())
}
```

<details>
<summary>✅ Solution</summary>

```rust
use std::error::Error;
use std::fs;

#[derive(Debug, PartialEq)]
struct FileStats {
    lines: usize,
    words: usize,
    chars: usize,
}

fn file_stats(path: &str) -> Result<FileStats, Box<dyn Error>> {
    let content = fs::read_to_string(path)?;

    let lines = content.lines().count();
    let words = content.split_whitespace().count();
    let chars = content.chars().count();

    Ok(FileStats { lines, words, chars })
}

fn main() -> Result<(), Box<dyn Error>> {
    // Create a test file
    fs::write("sample.txt", "Hello World\nRust is great\n")?;

    let stats = file_stats("sample.txt")?;
    assert_eq!(stats.lines, 2);
    assert_eq!(stats.words, 5);
    assert_eq!(stats.chars, 26);

    println!("{stats:?}");

    // Clean up
    fs::remove_file("sample.txt")?;
    Ok(())
}
```

</details>

### Exercise 2: Multi-Step Parser with Custom Error

Write a function that reads a file containing comma-separated `name,age` pairs (one per
line), parses them, and returns a `Vec<Person>`. Define a custom `ParseError` enum with
`From` implementations so that `?` works across error types.

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
}

#[derive(Debug)]
enum ParseError {
    Io(io::Error),
    InvalidAge(ParseIntError),
    InvalidFormat(String),
}

// TODO: Implement From<io::Error> for ParseError
// TODO: Implement From<ParseIntError> for ParseError

fn parse_people(path: &str) -> Result<Vec<Person>, ParseError> {
    // TODO:
    // 1. Read the file using fs::read_to_string (use ? — needs From<io::Error>)
    // 2. For each non-empty line, split on ',' 
    // 3. If there aren't exactly 2 parts, return InvalidFormat
    // 4. Parse the age as u32 (use ? — needs From<ParseIntError>)
    // 5. Collect into Vec<Person>
    todo!()
}

fn main() {
    match parse_people("people.csv") {
        Ok(people) => {
            for p in &people {
                println!("{} is {} years old", p.name, p.age);
            }
        }
        Err(e) => eprintln!("Error: {e:?}"),
    }
}
```

<details>
<summary>✅ Solution</summary>

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
}

#[derive(Debug)]
enum ParseError {
    Io(io::Error),
    InvalidAge(ParseIntError),
    InvalidFormat(String),
}

impl From<io::Error> for ParseError {
    fn from(err: io::Error) -> Self {
        ParseError::Io(err)
    }
}

impl From<ParseIntError> for ParseError {
    fn from(err: ParseIntError) -> Self {
        ParseError::InvalidAge(err)
    }
}

fn parse_people(path: &str) -> Result<Vec<Person>, ParseError> {
    let content = fs::read_to_string(path)?;    // io::Error → ParseError
    let mut people = Vec::new();

    for (i, line) in content.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() {
            continue;
        }

        let parts: Vec<&str> = line.splitn(2, ',').collect();
        if parts.len() != 2 {
            return Err(ParseError::InvalidFormat(
                format!("Line {}: expected 'name,age', got '{line}'", i + 1),
            ));
        }

        let name = parts[0].trim().to_string();
        let age: u32 = parts[1].trim().parse()?;  // ParseIntError → ParseError

        people.push(Person { name, age });
    }

    Ok(people)
}

fn main() {
    // Create test file
    fs::write("people.csv", "Alice, 30\nBob, 25\nCharlie, 35\n").unwrap();

    match parse_people("people.csv") {
        Ok(people) => {
            for p in &people {
                println!("{} is {} years old", p.name, p.age);
            }
        }
        Err(e) => eprintln!("Error: {e:?}"),
    }

    // Clean up
    let _ = fs::remove_file("people.csv");
}
```

</details>

### Exercise 3: Option Chains

Write a function `get_extension` that takes a file path as `&str` and returns
`Option<&str>` for the file extension. Use `?` with `Option` — do NOT use `if let` or
`match`.

```rust
fn get_extension(path: &str) -> Option<&str> {
    // TODO: Use ? with Option to:
    // 1. Find the last '.' in the path using rfind
    // 2. Check that it's not the first character (hidden files like .gitignore)
    // 3. Return the slice after the '.'
    // Hint: str::rfind returns Option<usize>
    todo!()
}

fn main() {
    assert_eq!(get_extension("photo.jpg"), Some("jpg"));
    assert_eq!(get_extension("archive.tar.gz"), Some("gz"));
    assert_eq!(get_extension("noext"), None);
    assert_eq!(get_extension(".gitignore"), None);
    println!("All assertions passed!");
}
```

<details>
<summary>✅ Solution</summary>

```rust
fn get_extension(path: &str) -> Option<&str> {
    let dot_pos = path.rfind('.')?;

    // If dot is at position 0, it's a hidden file — no extension
    if dot_pos == 0 {
        return None;
    }

    Some(&path[dot_pos + 1..])
}

fn main() {
    assert_eq!(get_extension("photo.jpg"), Some("jpg"));
    assert_eq!(get_extension("archive.tar.gz"), Some("gz"));
    assert_eq!(get_extension("noext"), None);
    assert_eq!(get_extension(".gitignore"), None);
    println!("All assertions passed!");
}
```

Here, `path.rfind('.')?` returns `None` immediately if there's no dot. The `?` replaces
a `match` or `if let` — clean and concise.

</details>

---

## Summary

| Concept | Key Point |
|---|---|
| **What `?` does** | Unwraps `Ok`/`Some`, returns `Err`/`None` early |
| **Desugaring** | `match expr { Ok(v) => v, Err(e) => return Err(From::from(e)) }` |
| **`From::from(e)`** | Automatic error type conversion — the secret ingredient |
| **Chaining** | `File::open(p)?.read_to_string(&mut buf)?` — postfix makes it natural |
| **With `Option`** | Works since Rust 1.22 — returns `None` early |
| **In `main()`** | `fn main() -> Result<(), Box<dyn Error>>` since Rust 1.26 |
| **History** | `try!` macro (2015) → `?` operator (2016) → `try!` deprecated (2018 edition) |
| **vs other languages** | Explicit like Go, concise like Python — best of both worlds |
| **When to use** | Whenever you'd write `match result { Ok(v) => v, Err(e) => return Err(e) }` |
| **When NOT to use** | When you need to handle the error locally (add context, retry, log, etc.) |

The `?` operator is one of Rust's crown jewels. It proves that safety and ergonomics aren't
at odds — you can have error handling that's **explicit**, **type-safe**, **zero-cost**, and
**beautiful** all at the same time. One character. Infinite elegance.

---

**Previous:** [← Result in Depth](./03-result-in-depth.md) · **Next:** [Custom Error Types →](./05-custom-error-types.md)

<p align="center"><i>Tutorial 4 of 7 — Stage 7: Error Handling</i></p>

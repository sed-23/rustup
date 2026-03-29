# Custom Error Types — Defining Your Own Errors 🎯

> **"When standard errors aren't enough, Rust lets you build precise, type-safe error types that make impossible states unrepresentable — and every failure self-documenting."**

---

## Table of Contents

- [Why Custom Error Types?](#why-custom-error-types)
- [The Error Trait Requirements](#the-error-trait-requirements)
- [Building a Custom Error Step by Step](#building-a-custom-error-step-by-step)
- [Enum Errors — The Idiomatic Approach](#enum-errors--the-idiomatic-approach)
- [The From Trait — Making ? Work with Your Errors](#the-from-trait--making--work-with-your-errors)
- [Error Wrapping and Chaining](#error-wrapping-and-chaining)
- [Designing Good Error Types](#designing-good-error-types)
- [Custom Errors in Other Languages](#custom-errors-in-other-languages)
- [Real-World Example: A Config Parser Error Type](#real-world-example-a-config-parser-error-type)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Why Custom Error Types?

The standard library gives you errors like `io::Error`, `ParseIntError`, and `fmt::Error`. These are fine for low-level operations, but they fall apart the moment your application grows beyond a single function.

Consider a web server. When a request fails, you need to know *why*:

- Was the user not authenticated? → `AuthError`
- Did the database connection drop? → `DatabaseError`
- Was the input malformed? → `ValidationError`
- Did a background task time out? → `TimeoutError`

Returning `io::Error` for all of these is like a doctor diagnosing every patient with "you're sick." Technically true, utterly useless.

Custom error types give you:

| Benefit | Description |
|---|---|
| **Precision** | Each variant describes exactly what went wrong |
| **Pattern matching** | Callers can `match` on error kinds and handle each one differently |
| **Context** | You can attach relevant data — file paths, user IDs, query strings |
| **Type safety** | The compiler ensures you handle every possible error variant |
| **Composability** | `From` implementations let `?` convert foreign errors into yours automatically |

By the end of this tutorial, you'll be able to define production-quality error types from scratch — the same patterns used by `serde`, `reqwest`, `diesel`, and virtually every serious Rust crate.

---

## The Error Trait Requirements

Before building anything, let's understand what Rust expects from an error type. The `std::error::Error` trait is defined (simplified) as:

```rust
pub trait Error: fmt::Display + fmt::Debug {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

That's it. Three requirements, one optional:

### 1. `fmt::Debug` — The Debug Representation

This is almost always derived automatically:

```rust
#[derive(Debug)]
struct MyError {
    message: String,
}
```

`Debug` produces the programmer-facing output you see with `{:?}`. It's used in panic messages, test failures, and `.unwrap()` output.

### 2. `fmt::Display` — The Human-Readable Message

This must be implemented manually. It's what the user (or log file) sees:

```rust
use std::fmt;

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}
```

`Display` is used by `println!("{}", error)`, `.to_string()`, and the default `Error` trait output.

### 3. `std::error::Error` — The Trait Itself

The simplest implementation is an empty `impl` block, since `source()` has a default:

```rust
impl std::error::Error for MyError {}
```

### 4. `source()` — Optional Error Chaining

If your error wraps another error (e.g., your `DatabaseError` wraps an `io::Error`), implement `source()` to expose the underlying cause:

```rust
impl std::error::Error for MyError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        Some(&self.inner_error) // return the wrapped error
    }
}
```

This enables error chain traversal — walking from the top-level error down to the root cause.

---

## Building a Custom Error Step by Step

Let's build a complete custom error from nothing, one piece at a time.

### Step 1: Define the Struct

```rust
struct AppError {
    message: String,
}
```

Right now this is just a normal struct. It can't be used as an error — you can't return it from a function that returns `Result`, you can't print it, and you can't use `?` with it.

### Step 2: Derive Debug

```rust
#[derive(Debug)]
struct AppError {
    message: String,
}
```

Now `{:?}` works:

```rust
let e = AppError { message: "something broke".into() };
println!("{:?}", e);
// AppError { message: "something broke" }
```

### Step 3: Implement Display

```rust
use std::fmt;

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Application error: {}", self.message)
    }
}
```

Now `{}` works:

```rust
println!("{}", e);
// Application error: something broke
```

### Step 4: Implement the Error Trait

```rust
impl std::error::Error for AppError {}
```

Now `AppError` is a proper Rust error type. You can:

```rust
fn do_something() -> Result<(), AppError> {
    Err(AppError {
        message: "disk full".into(),
    })
}

fn main() {
    match do_something() {
        Ok(()) => println!("Success!"),
        Err(e) => eprintln!("Failed: {e}"),
    }
}
// Failed: Application error: disk full
```

### The Full Boilerplate

Here's everything in one place — the minimum viable custom error:

```rust
use std::fmt;

#[derive(Debug)]
struct AppError {
    message: String,
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Application error: {}", self.message)
    }
}

impl std::error::Error for AppError {}

impl AppError {
    fn new(msg: &str) -> Self {
        AppError {
            message: msg.to_string(),
        }
    }
}

fn main() -> Result<(), AppError> {
    Err(AppError::new("something went wrong"))
}
```

This works, but struct-based errors are limited. You have one shape for every failure. In practice, you almost always want an **enum**.

---

## Enum Errors — The Idiomatic Approach

Most Rust errors are **enums**. Each variant represents a different failure mode, and each variant can carry different data.

### Defining the Enum

```rust
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    NotFound(String),
    PermissionDenied,
    DatabaseError(String),
    ParseError(ParseIntError),
}
```

Four variants, four different kinds of failure. Some carry a `String` for context, one carries the original `ParseIntError` for chaining.

### Implementing Display

Match on each variant and write a clear message:

```rust
use std::fmt;

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(resource) => {
                write!(f, "Resource not found: {resource}")
            }
            AppError::PermissionDenied => {
                write!(f, "Permission denied")
            }
            AppError::DatabaseError(details) => {
                write!(f, "Database error: {details}")
            }
            AppError::ParseError(e) => {
                write!(f, "Parse error: {e}")
            }
        }
    }
}
```

### Implementing Error with source()

For variants that wrap another error, return it from `source()`:

```rust
impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::ParseError(e) => Some(e),
            _ => None,
        }
    }
}
```

Only `ParseError` wraps a foreign error, so only that variant returns `Some`. The rest return `None` (via the catch-all `_`).

### Using the Enum Error

```rust
fn find_user(id: &str) -> Result<String, AppError> {
    if id.is_empty() {
        return Err(AppError::NotFound("user ID is empty".into()));
    }

    // Simulate a permission check
    if id == "admin" {
        return Err(AppError::PermissionDenied);
    }

    Ok(format!("User({id})"))
}

fn main() {
    match find_user("admin") {
        Ok(user) => println!("Found: {user}"),
        Err(AppError::PermissionDenied) => {
            eprintln!("Access denied — you are not authorized.");
        }
        Err(e) => eprintln!("Error: {e}"),
    }
}
// Access denied — you are not authorized.
```

The caller can match on **specific variants** and handle them differently. This is impossible with a plain `String` or a generic `io::Error`.

---

## The From Trait — Making ? Work with Your Errors

The `?` operator does two things:

1. If the `Result` is `Ok`, unwrap the value and continue.
2. If the `Result` is `Err(e)`, call `From::from(e)` to convert the error, then return early.

Step 2 is the key. If your function returns `Result<T, AppError>`, and you call a function that returns `Result<T, io::Error>`, the `?` operator needs a way to convert `io::Error` into `AppError`. That's what `From` provides.

### Implementing From for io::Error

```rust
use std::io;

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::DatabaseError(e.to_string())
    }
}
```

### Implementing From for ParseIntError

```rust
use std::num::ParseIntError;

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self {
        AppError::ParseError(e)
    }
}
```

### Now ? Just Works

```rust
use std::fs;

fn read_age_from_file(path: &str) -> Result<u32, AppError> {
    let contents = fs::read_to_string(path)?;   // io::Error → AppError
    let age: u32 = contents.trim().parse()?;     // ParseIntError → AppError
    Ok(age)
}

fn main() {
    match read_age_from_file("age.txt") {
        Ok(age) => println!("Age: {age}"),
        Err(AppError::ParseError(e)) => {
            eprintln!("File contained invalid number: {e}");
        }
        Err(e) => eprintln!("Failed: {e}"),
    }
}
```

Two different error types (`io::Error` and `ParseIntError`), both converted seamlessly by `?` thanks to the `From` implementations. No manual matching, no `.map_err()` clutter.

### The Full Picture

Here's a diagram of what happens when `?` is used:

```
fs::read_to_string(path)?
         │
         ├── Ok(contents) → unwrap, continue
         │
         └── Err(io::Error) → From::from(io::Error) → AppError::DatabaseError(...)
                                                          │
                                                          └── return Err(AppError::DatabaseError(...))
```

Every `From` impl you write unlocks one more error type for `?` conversion. This is why `From` is the engine behind Rust's ergonomic error handling.

---

## Error Wrapping and Chaining

When your custom error wraps another error, you create an **error chain** — a linked list of errors from the top-level failure down to the root cause.

### Storing the Original Error

Instead of converting to a `String` (which loses the source), store the original error:

```rust
#[derive(Debug)]
enum AppError {
    Io(std::io::Error),                     // wraps io::Error directly
    Parse(std::num::ParseIntError),         // wraps ParseIntError directly
    Config { key: String, source: Box<dyn std::error::Error> },
}
```

### Implementing source() for Chaining

```rust
impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::Io(e) => Some(e),
            AppError::Parse(e) => Some(e),
            AppError::Config { source, .. } => Some(source.as_ref()),
        }
    }
}
```

### Walking the Error Chain

You can iterate through `source()` to print every error in the chain:

```rust
fn print_error_chain(error: &dyn std::error::Error) {
    println!("Error: {error}");

    let mut current = error.source();
    let mut depth = 1;

    while let Some(cause) = current {
        println!("  {depth}: Caused by: {cause}");
        current = cause.source();
        depth += 1;
    }
}
```

### Example Output

```rust
fn load_port() -> Result<u16, AppError> {
    let contents = std::fs::read_to_string("config.txt")
        .map_err(AppError::Io)?;

    let port: u16 = contents.trim().parse()
        .map_err(AppError::Parse)?;

    Ok(port)
}

fn main() {
    match load_port() {
        Ok(port) => println!("Port: {port}"),
        Err(ref e) => print_error_chain(e),
    }
}
```

If `config.txt` contains `"abc"`, the output is:

```
Error: Parse error: invalid digit found in string
  1: Caused by: invalid digit found in string
```

If `config.txt` doesn't exist:

```
Error: IO error: No such file or directory (os error 2)
  1: Caused by: No such file or directory (os error 2)
```

The chain gives you the full story — your domain error *plus* the underlying system error.

### Why Chaining Matters

Without chaining, you lose context:

```rust
// Bad: converts to String, loses the original error
AppError::DatabaseError(e.to_string())

// Good: preserves the original error for chaining
AppError::Io(e)
```

Libraries like `anyhow` and `thiserror` (covered in the next tutorial) automate this pattern, but understanding the manual approach is essential for knowing what they do under the hood.

---

## Designing Good Error Types

Writing a custom error is easy. Designing a *good* one takes thought. Here are the principles used by well-maintained Rust crates.

### One Error Enum per Module or Crate

Don't create a single `AppError` with 50 variants for your entire application. Instead:

```
src/
  auth/
    mod.rs      → defines AuthError
  db/
    mod.rs      → defines DbError
  api/
    mod.rs      → defines ApiError (may wrap AuthError, DbError)
```

Each module owns its error type. The top-level `ApiError` wraps the lower-level ones:

```rust
#[derive(Debug)]
enum ApiError {
    Auth(AuthError),
    Db(DbError),
    BadRequest(String),
}
```

### Variants Should Carry Useful Context

Don't do this:

```rust
enum AppError {
    IoError,           // What file? What operation? Useless.
    ConfigError,       // Which key? What value? No clue.
}
```

Do this:

```rust
enum AppError {
    IoError {
        path: std::path::PathBuf,
        operation: String,
        source: std::io::Error,
    },
    ConfigError {
        key: String,
        expected: String,
        found: String,
    },
}
```

Now the error message can say: *"Failed to read /etc/app/config.toml: permission denied"* instead of *"IO error"*.

### Don't Expose Implementation Details

If your crate uses `sqlite` internally, don't put `SqliteError` in your public error type. Users shouldn't need to depend on `sqlite` just to handle your errors.

```rust
// Bad: leaks internal dependency
pub enum MyError {
    Sqlite(rusqlite::Error),  // forces users to depend on rusqlite
}

// Good: hides the implementation
pub enum MyError {
    Storage {
        message: String,
        source: Box<dyn std::error::Error + Send + Sync>,
    },
}
```

### Use Box\<dyn Error\> for Application Code

Libraries should define precise error types. Applications (binaries) can be more relaxed:

```rust
// In a library: be precise
pub fn parse_config(path: &str) -> Result<Config, ConfigError> { ... }

// In an application: Box<dyn Error> is fine
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = parse_config("app.toml")?;
    let db = connect_database(&config)?;
    run_server(db)?;
    Ok(())
}
```

`Box<dyn Error>` accepts *any* error type. It's the "I don't care what went wrong, just print it" approach — perfect for `main()` and scripts, inappropriate for libraries.

### Summary of Guidelines

| Guideline | Reason |
|---|---|
| One error type per module | Keeps errors focused and maintainable |
| Carry context in variants | Makes error messages actionable |
| Use `source()` for chaining | Preserves the root cause |
| Hide implementation details | Avoids leaking internal dependencies |
| `Box<dyn Error>` for apps | Simplifies top-level error handling |
| Precise types for libraries | Lets consumers match on specific failures |

---

## Custom Errors in Other Languages

Rust's approach to custom errors is unique. Let's see how other languages handle the same problem.

### Java: Exception Class Hierarchy

```java
public class AuthException extends Exception {
    private final String userId;

    public AuthException(String userId, String message) {
        super(message);
        this.userId = userId;
    }

    public String getUserId() {
        return userId;
    }
}

// Usage
try {
    authenticate(user);
} catch (AuthException e) {
    System.out.println("Auth failed for: " + e.getUserId());
}
```

Java forces you into a class hierarchy. You must choose between checked exceptions (must be declared) and unchecked exceptions (can be ignored). The hierarchy is rigid and verbose.

### Python: Custom Exception Classes

```python
class AuthError(Exception):
    def __init__(self, user_id, message):
        super().__init__(message)
        self.user_id = user_id

# Usage
try:
    authenticate(user)
except AuthError as e:
    print(f"Auth failed for {e.user_id}: {e}")
```

Python is flexible but offers no compile-time guarantees. You can forget to catch an exception entirely, and nothing warns you.

### Go: Error Interface

```go
type AuthError struct {
    UserID  string
    Message string
}

func (e *AuthError) Error() string {
    return fmt.Sprintf("auth failed for %s: %s", e.UserID, e.Message)
}

// Usage
if err != nil {
    var authErr *AuthError
    if errors.As(err, &authErr) {
        fmt.Println("User:", authErr.UserID)
    }
}
```

Go's approach is simple (just implement `Error() string`) but loses type information. You need runtime type assertions (`errors.As`) to inspect specific error kinds.

### C++: Exception Classes

```cpp
class AuthError : public std::runtime_error {
    std::string user_id_;
public:
    AuthError(const std::string& user_id, const std::string& msg)
        : std::runtime_error(msg), user_id_(user_id) {}

    const std::string& user_id() const { return user_id_; }
};

// Usage
try {
    authenticate(user);
} catch (const AuthError& e) {
    std::cout << "Auth failed for: " << e.user_id() << "\n";
}
```

C++ exceptions unwind the stack, can throw from anywhere, and have complex performance characteristics. They're powerful but hard to reason about.

### Comparison Table

| Feature | Rust | Java | Python | Go | C++ |
|---|---|---|---|---|---|
| **Mechanism** | Enum + Result | Class hierarchy | Class hierarchy | Struct + interface | Class hierarchy |
| **Compile-time checking** | ✅ Must handle Result | ✅ Checked exceptions | ❌ None | ❌ None | ❌ None |
| **Pattern matching** | ✅ match on variants | ⚠️ catch blocks | ⚠️ except blocks | ❌ Type assertions | ⚠️ catch blocks |
| **Stack unwinding** | ❌ No (values) | ✅ Yes | ✅ Yes | ❌ No (values) | ✅ Yes |
| **Error chaining** | ✅ source() | ✅ getCause() | ✅ \_\_cause\_\_ | ✅ errors.Unwrap() | ❌ Manual |
| **Closest analogue** | Haskell/OCaml sum types | — | — | — | — |
| **Boilerplate** | Medium (or use thiserror) | High | Low | Low | High |

Rust's enum-based errors are most similar to **algebraic sum types** in Haskell (`data AppError = NotFound String | PermDenied | ...`) and OCaml (`type error = NotFound of string | PermDenied | ...`). The key advantage over all of the above: the compiler *forces* you to handle every variant.

---

## Real-World Example: A Config Parser Error Type

Let's build a complete, production-quality error type for a configuration file parser. This ties together everything from this tutorial.

### The Scenario

We're parsing a config file like:

```
host=localhost
port=8080
max_connections=100
```

Things that can go wrong:
- File doesn't exist or can't be read (`io::Error`)
- A line is malformed — no `=` sign
- A required key is missing
- A value can't be parsed as the expected type (`ParseIntError`)

### The Error Type

```rust
use std::fmt;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    /// Failed to read the config file
    Io {
        path: String,
        source: io::Error,
    },
    /// A line in the config file is malformed
    MalformedLine {
        line_number: usize,
        content: String,
    },
    /// A required configuration key is missing
    MissingKey {
        key: String,
    },
    /// A value could not be parsed as the expected type
    InvalidValue {
        key: String,
        value: String,
        source: ParseIntError,
    },
}
```

### Display Implementation

```rust
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Io { path, source } => {
                write!(f, "Failed to read config file '{path}': {source}")
            }
            ConfigError::MalformedLine { line_number, content } => {
                write!(
                    f,
                    "Malformed config at line {line_number}: '{content}'"
                )
            }
            ConfigError::MissingKey { key } => {
                write!(f, "Missing required config key: '{key}'")
            }
            ConfigError::InvalidValue { key, value, source } => {
                write!(
                    f,
                    "Invalid value for '{key}': '{value}' ({source})"
                )
            }
        }
    }
}
```

### Error Trait with source()

```rust
impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::Io { source, .. } => Some(source),
            ConfigError::InvalidValue { source, .. } => Some(source),
            _ => None,
        }
    }
}
```

### From Implementations

```rust
impl From<io::Error> for ConfigError {
    fn from(e: io::Error) -> Self {
        ConfigError::Io {
            path: String::from("<unknown>"),
            source: e,
        }
    }
}
```

> **Note:** The basic `From<io::Error>` uses `"<unknown>"` for the path because `From` only takes the source error — no extra context. For better messages, use `.map_err()` to add the path explicitly.

### The Config Parser

```rust
use std::collections::HashMap;
use std::fs;

struct Config {
    host: String,
    port: u16,
    max_connections: u32,
}

fn parse_config(path: &str) -> Result<Config, ConfigError> {
    // Read file — io::Error converted via map_err for better context
    let contents = fs::read_to_string(path).map_err(|e| ConfigError::Io {
        path: path.to_string(),
        source: e,
    })?;

    // Parse key-value pairs
    let mut map = HashMap::new();

    for (i, line) in contents.lines().enumerate() {
        let line = line.trim();

        // Skip empty lines and comments
        if line.is_empty() || line.starts_with('#') {
            continue;
        }

        let (key, value) = line
            .split_once('=')
            .ok_or(ConfigError::MalformedLine {
                line_number: i + 1,
                content: line.to_string(),
            })?;

        map.insert(key.trim().to_string(), value.trim().to_string());
    }

    // Extract required keys
    let host = map
        .remove("host")
        .ok_or(ConfigError::MissingKey {
            key: "host".to_string(),
        })?;

    let port_str = map
        .get("port")
        .ok_or(ConfigError::MissingKey {
            key: "port".to_string(),
        })?;

    let port: u16 = port_str.parse().map_err(|e| ConfigError::InvalidValue {
        key: "port".to_string(),
        value: port_str.clone(),
        source: e,
    })?;

    let max_str = map
        .get("max_connections")
        .ok_or(ConfigError::MissingKey {
            key: "max_connections".to_string(),
        })?;

    let max_connections: u32 =
        max_str
            .parse()
            .map_err(|e| ConfigError::InvalidValue {
                key: "max_connections".to_string(),
                value: max_str.clone(),
                source: e,
            })?;

    Ok(Config {
        host,
        port,
        max_connections,
    })
}
```

### Using It

```rust
fn print_error_chain(error: &dyn std::error::Error) {
    eprintln!("Error: {error}");
    let mut source = error.source();
    let mut i = 1;
    while let Some(cause) = source {
        eprintln!("  {i}: Caused by: {cause}");
        source = cause.source();
        i += 1;
    }
}

fn main() {
    match parse_config("server.conf") {
        Ok(config) => {
            println!("Host: {}", config.host);
            println!("Port: {}", config.port);
            println!("Max connections: {}", config.max_connections);
        }
        Err(ref e) => print_error_chain(e),
    }
}
```

### Sample Output for Different Failures

**File doesn't exist:**
```
Error: Failed to read config file 'server.conf': No such file or directory (os error 2)
  1: Caused by: No such file or directory (os error 2)
```

**Malformed line (e.g., `port 8080` without `=`):**
```
Error: Malformed config at line 2: 'port 8080'
```

**Invalid value (e.g., `port=abc`):**
```
Error: Invalid value for 'port': 'abc' (invalid digit found in string)
  1: Caused by: invalid digit found in string
```

**Missing key:**
```
Error: Missing required config key: 'max_connections'
```

Every failure is specific, actionable, and chainable. This is the standard pattern for error types in the Rust ecosystem.

---

## Exercises

### Exercise 1: Build a File Processor Error Type

Create a `FileProcessorError` enum that handles these failure cases:

- File not found (wraps `io::Error`, stores the file path)
- Empty file (stores the file path)
- Line too long (stores line number, max allowed length, actual length)
- Invalid encoding (stores a description)

Implement `Debug`, `Display`, `Error` (with `source()`), and `From<io::Error>`.

Write a function `process_file(path: &str) -> Result<Vec<String>, FileProcessorError>` that:
1. Reads the file (use `?` — the `From` impl converts `io::Error`)
2. Returns `EmptyFile` if the file has no lines
3. Returns `LineTooLong` if any line exceeds 120 characters
4. Otherwise returns all lines as a `Vec<String>`

<details>
<summary>💡 Solution</summary>

```rust
use std::fmt;
use std::fs;
use std::io;

#[derive(Debug)]
enum FileProcessorError {
    FileNotFound {
        path: String,
        source: io::Error,
    },
    EmptyFile {
        path: String,
    },
    LineTooLong {
        line_number: usize,
        max_length: usize,
        actual_length: usize,
    },
    InvalidEncoding {
        description: String,
    },
}

impl fmt::Display for FileProcessorError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            FileProcessorError::FileNotFound { path, source } => {
                write!(f, "File not found '{path}': {source}")
            }
            FileProcessorError::EmptyFile { path } => {
                write!(f, "File is empty: '{path}'")
            }
            FileProcessorError::LineTooLong {
                line_number,
                max_length,
                actual_length,
            } => {
                write!(
                    f,
                    "Line {line_number} too long: {actual_length} chars \
                     (max {max_length})"
                )
            }
            FileProcessorError::InvalidEncoding { description } => {
                write!(f, "Invalid encoding: {description}")
            }
        }
    }
}

impl std::error::Error for FileProcessorError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            FileProcessorError::FileNotFound { source, .. } => Some(source),
            _ => None,
        }
    }
}

impl From<io::Error> for FileProcessorError {
    fn from(e: io::Error) -> Self {
        FileProcessorError::FileNotFound {
            path: String::from("<unknown>"),
            source: e,
        }
    }
}

fn process_file(path: &str) -> Result<Vec<String>, FileProcessorError> {
    let contents = fs::read_to_string(path).map_err(|e| {
        FileProcessorError::FileNotFound {
            path: path.to_string(),
            source: e,
        }
    })?;

    let lines: Vec<String> = contents.lines().map(String::from).collect();

    if lines.is_empty() {
        return Err(FileProcessorError::EmptyFile {
            path: path.to_string(),
        });
    }

    const MAX_LENGTH: usize = 120;
    for (i, line) in lines.iter().enumerate() {
        if line.len() > MAX_LENGTH {
            return Err(FileProcessorError::LineTooLong {
                line_number: i + 1,
                max_length: MAX_LENGTH,
                actual_length: line.len(),
            });
        }
    }

    Ok(lines)
}

fn main() {
    match process_file("data.txt") {
        Ok(lines) => println!("Processed {} lines", lines.len()),
        Err(e) => eprintln!("{e}"),
    }
}
```

</details>

---

### Exercise 2: Error Chain Walker

Write a function `format_error_report(error: &dyn std::error::Error) -> String` that produces a formatted multi-line report like:

```
[ERROR] Failed to read config file 'app.toml': permission denied
  └─ caused by: permission denied (os error 13)
```

If there are multiple levels:

```
[ERROR] Application startup failed
  └─ caused by: Failed to read config file 'app.toml': permission denied
      └─ caused by: permission denied (os error 13)
```

Test it with your error types from Exercise 1.

<details>
<summary>💡 Solution</summary>

```rust
fn format_error_report(error: &dyn std::error::Error) -> String {
    let mut report = format!("[ERROR] {error}\n");
    let mut current = error.source();
    let mut depth = 1;

    while let Some(cause) = current {
        let indent = "    ".repeat(depth - 1);
        report.push_str(&format!("{indent}  └─ caused by: {cause}\n"));
        current = cause.source();
        depth += 1;
    }

    report
}

// Test it
fn main() {
    let inner = std::io::Error::new(
        std::io::ErrorKind::PermissionDenied,
        "permission denied",
    );

    let error = FileProcessorError::FileNotFound {
        path: "app.toml".to_string(),
        source: inner,
    };

    print!("{}", format_error_report(&error));
}

// Output:
// [ERROR] File not found 'app.toml': permission denied
//   └─ caused by: permission denied
```

</details>

---

### Exercise 3: Multi-Module Error Composition

Create two error enums — `AuthError` and `DbError` — and a top-level `AppError` that wraps both. Implement `From<AuthError> for AppError` and `From<DbError> for AppError`.

Write functions:
- `authenticate(user: &str) -> Result<(), AuthError>` — fails if user is `"banned"`
- `fetch_data(user: &str) -> Result<String, DbError>` — fails if user is `"unknown"`
- `handle_request(user: &str) -> Result<String, AppError>` — calls both, using `?`

<details>
<summary>💡 Solution</summary>

```rust
use std::fmt;

// --- Auth module ---
#[derive(Debug)]
enum AuthError {
    Banned { user: String },
    Expired { user: String },
}

impl fmt::Display for AuthError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AuthError::Banned { user } => write!(f, "User '{user}' is banned"),
            AuthError::Expired { user } => {
                write!(f, "Session expired for '{user}'")
            }
        }
    }
}

impl std::error::Error for AuthError {}

// --- Database module ---
#[derive(Debug)]
enum DbError {
    NotFound { user: String },
    ConnectionFailed { details: String },
}

impl fmt::Display for DbError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DbError::NotFound { user } => {
                write!(f, "No data found for user '{user}'")
            }
            DbError::ConnectionFailed { details } => {
                write!(f, "DB connection failed: {details}")
            }
        }
    }
}

impl std::error::Error for DbError {}

// --- Top-level application error ---
#[derive(Debug)]
enum AppError {
    Auth(AuthError),
    Db(DbError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Auth(e) => write!(f, "Authentication error: {e}"),
            AppError::Db(e) => write!(f, "Database error: {e}"),
        }
    }
}

impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::Auth(e) => Some(e),
            AppError::Db(e) => Some(e),
        }
    }
}

impl From<AuthError> for AppError {
    fn from(e: AuthError) -> Self {
        AppError::Auth(e)
    }
}

impl From<DbError> for AppError {
    fn from(e: DbError) -> Self {
        AppError::Db(e)
    }
}

// --- Functions ---
fn authenticate(user: &str) -> Result<(), AuthError> {
    if user == "banned" {
        return Err(AuthError::Banned {
            user: user.to_string(),
        });
    }
    Ok(())
}

fn fetch_data(user: &str) -> Result<String, DbError> {
    if user == "unknown" {
        return Err(DbError::NotFound {
            user: user.to_string(),
        });
    }
    Ok(format!("Data for {user}"))
}

fn handle_request(user: &str) -> Result<String, AppError> {
    authenticate(user)?;     // AuthError → AppError via From
    let data = fetch_data(user)?;  // DbError → AppError via From
    Ok(data)
}

fn main() {
    for user in &["alice", "banned", "unknown"] {
        match handle_request(user) {
            Ok(data) => println!("{user}: {data}"),
            Err(e) => eprintln!("{user}: {e}"),
        }
    }
}

// Output:
// alice: Data for alice
// banned: Authentication error: User 'banned' is banned
// unknown: Database error: No data found for user 'unknown'
```

</details>

---

## Summary

| Concept | What You Learned |
|---|---|
| **Why custom errors** | Standard library errors are too generic for real applications |
| **Error trait** | Requires `Debug` + `Display`, optionally `source()` for chaining |
| **Struct errors** | Simple but limited — one shape for all failures |
| **Enum errors** | Idiomatic Rust — each variant is a different failure kind |
| **`From` trait** | Enables `?` to automatically convert foreign errors into yours |
| **Error chaining** | Store original errors, expose them via `source()` |
| **Design principles** | One type per module, carry context, hide internals |
| **Cross-language** | Rust's approach is closest to algebraic sum types (Haskell, OCaml) |

The manual approach shown here works perfectly, but it's verbose. In the next tutorial, we'll see how `thiserror` and `anyhow` eliminate the boilerplate while preserving all the type safety and chaining you've learned here.

---

**Previous:** [← The ? Operator](./04-question-mark-operator.md) · **Next:** [thiserror & anyhow →](./06-thiserror-and-anyhow.md)

<p align="center"><i>Tutorial 5 of 7 — Stage 7: Error Handling</i></p>

# Error Handling Best Practices — Real-World Patterns 🎯

> **"Good error handling is invisible when things go right and invaluable when things go wrong."**

---

## Table of Contents

- [The Error Handling Decision Tree](#the-error-handling-decision-tree)
- [Pattern: The Error Conversion Layer](#pattern-the-error-conversion-layer)
- [Pattern: Error Context Enrichment](#pattern-error-context-enrichment)
- [Pattern: Structured Error Logging](#pattern-structured-error-logging)
- [Pattern: Graceful Degradation](#pattern-graceful-degradation)
- [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
- [Error Handling in Different Contexts](#error-handling-in-different-contexts)
- [Testing Error Paths](#testing-error-paths)
- [Real-World Case Study: A File Processor](#real-world-case-study-a-file-processor)
- [The Rust Community's Error Handling Guidelines](#the-rust-communitys-error-handling-guidelines)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Error Handling Decision Tree

Every time you write a function that can fail, you face a series of decisions. After six
tutorials of building up error handling tools, here's how to pick the right one.

### Question 1: Is This a Bug or an Expected Failure?

| Situation | Mechanism | Example |
|-----------|-----------|---------|
| **Bug** — should never happen if code is correct | `panic!`, `unreachable!`, `assert!` | Index out of bounds on a vec you just built |
| **Expected failure** — the outside world is messy | `Result<T, E>` | File doesn't exist, network timeout, invalid user input |

```rust
// BUG — this means our logic is broken, panic is appropriate
fn get_player(players: &[Player], index: usize) -> &Player {
    assert!(index < players.len(), "Player index out of bounds — logic error");
    &players[index]
}

// EXPECTED FAILURE — the user might give us garbage
fn parse_age(input: &str) -> Result<u8, String> {
    input.parse::<u8>().map_err(|e| format!("Invalid age '{}': {}", input, e))
}
```

### Question 2: Am I Writing a Library or an Application?

| Context | Recommended Crate | Why |
|---------|-------------------|-----|
| **Library** | `thiserror` | Callers need typed errors they can match on |
| **Application** | `anyhow` | You just need to report errors to the user |
| **Both** (lib + binary) | `thiserror` for lib, `anyhow` in `main.rs` | Best of both worlds |

### Question 3: Does the Caller Need to Match on Error Variants?

| Caller's Need | Error Type | Example |
|----------------|-----------|---------|
| **Yes** — caller branches on error kind | Custom `enum` with `thiserror` | Retry on timeout, abort on auth failure |
| **No** — caller just propagates or prints | `anyhow::Error` or `Box<dyn Error>` | CLI tool printing the error and exiting |

### The Decision Flowchart

```
                    ┌─────────────────────┐
                    │  Can this operation  │
                    │       fail?          │
                    └─────────┬───────────┘
                              │
                     ┌────────┴────────┐
                     │ Yes             │ No
                     ▼                 ▼
              ┌─────────────┐   Just return T
              │ Bug or       │
              │ expected?    │
              └──────┬──────┘
                     │
            ┌────────┴────────┐
            │ Bug             │ Expected
            ▼                 ▼
     panic!/assert!    ┌──────────────┐
                       │ Library or   │
                       │ application? │
                       └──────┬──────┘
                              │
                    ┌─────────┴─────────┐
                    │ Library           │ Application
                    ▼                   ▼
            ┌──────────────┐    ┌──────────────┐
            │ Caller needs │    │   anyhow     │
            │ to match?    │    │ (just report │
            └──────┬──────┘    │  errors)     │
                   │            └──────────────┘
          ┌────────┴────────┐
          │ Yes             │ No
          ▼                 ▼
   thiserror enum    Box<dyn Error>
   with variants     or anyhow
```

### Quick Reference Card

```rust
// Prototype / script         → .unwrap() / .expect("reason")
// Application binary         → anyhow::Result<T>
// Library crate              → thiserror enum
// Performance-critical path  → custom error, no allocation
// FFI boundary               → integer error codes
```

---

## Pattern: The Error Conversion Layer

In any real application, you deal with errors from many libraries — `std::io`,
`serde_json`, `reqwest`, `sqlx`, and more. Exposing those raw library errors to
your callers leaks implementation details. The fix: **convert at the boundary**.

### The Problem

```rust
// BAD: Your public API exposes io::Error, serde_json::Error, etc.
pub fn load_config(path: &str) -> Result<Config, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string(path)?;  // io::Error
    let config: Config = serde_json::from_str(&content)?;  // serde_json::Error
    Ok(config)
}
// Caller has NO IDEA what errors to expect. Box<dyn Error> is opaque.
```

### The Solution: Domain Error Types

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("Config file not found: {path}")]
    FileNotFound {
        path: String,
        #[source]
        source: std::io::Error,
    },

    #[error("Failed to read config file: {path}")]
    ReadError {
        path: String,
        #[source]
        source: std::io::Error,
    },

    #[error("Invalid config format in {path}")]
    ParseError {
        path: String,
        #[source]
        source: serde_json::Error,
    },

    #[error("Missing required field: {field}")]
    MissingField { field: String },
}

pub fn load_config(path: &str) -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string(path).map_err(|e| {
        if e.kind() == std::io::ErrorKind::NotFound {
            ConfigError::FileNotFound {
                path: path.to_string(),
                source: e,
            }
        } else {
            ConfigError::ReadError {
                path: path.to_string(),
                source: e,
            }
        }
    })?;

    let config: Config = serde_json::from_str(&content).map_err(|e| {
        ConfigError::ParseError {
            path: path.to_string(),
            source: e,
        }
    })?;

    if config.database_url.is_empty() {
        return Err(ConfigError::MissingField {
            field: "database_url".to_string(),
        });
    }

    Ok(config)
}
```

Now the caller can match on exactly what went wrong:

```rust
match load_config("app.json") {
    Ok(config) => start_app(config),
    Err(ConfigError::FileNotFound { path, .. }) => {
        eprintln!("Config file missing at '{}'. Creating default...", path);
        create_default_config(path)?;
    }
    Err(ConfigError::ParseError { path, source, .. }) => {
        eprintln!("Syntax error in '{}': {}", path, source);
        std::process::exit(1);
    }
    Err(e) => {
        eprintln!("Config error: {}", e);
        std::process::exit(1);
    }
}
```

### Rule of Thumb

> Convert library errors into **your** domain errors at **module boundaries**.
> Inside a module, feel free to use `?` with raw library errors.
> At the module's public API, expose only your own error type.

---

## Pattern: Error Context Enrichment

The single most common error handling mistake: **bare errors with no context**.

### The Problem

```
Error: No such file or directory (os error 2)
```

Which file? Which operation? Which part of the program? This error message is
almost useless in a real application.

### Solution 1: Manual Context with `map_err`

```rust
use std::fs;
use std::path::Path;

fn read_user_config(user_id: u64) -> Result<String, AppError> {
    let path = format!("/etc/myapp/users/{}.toml", user_id);

    fs::read_to_string(&path).map_err(|source| AppError::Config {
        message: format!("Failed to read config for user {}", user_id),
        path: path.clone(),
        source,
    })
}
```

Now the error says: `Failed to read config for user 42: /etc/myapp/users/42.toml: No such file or directory`

### Solution 2: Using `anyhow::Context`

```rust
use anyhow::{Context, Result};
use std::fs;

fn read_user_config(user_id: u64) -> Result<String> {
    let path = format!("/etc/myapp/users/{}.toml", user_id);

    fs::read_to_string(&path)
        .with_context(|| format!("Failed to read config for user {} at '{}'", user_id, path))
}
```

### Before and After

```
BEFORE (bare error):
  Error: No such file or directory (os error 2)

AFTER (with context):
  Error: Failed to read config for user 42 at '/etc/myapp/users/42.toml'

  Caused by:
      No such file or directory (os error 2)
```

### Stacking Context

Context composes — each `?` can add another layer:

```rust
use anyhow::{Context, Result};

fn process_batch(batch_id: u64) -> Result<()> {
    let config = load_config()
        .context("Failed to load application config")?;

    let input_path = config.input_dir.join(format!("batch_{}.csv", batch_id));

    let records = read_csv(&input_path)
        .with_context(|| format!("Failed to read batch {}", batch_id))?;

    validate_records(&records)
        .with_context(|| format!("Validation failed for batch {} ({} records)", batch_id, records.len()))?;

    Ok(())
}
```

Error output with full chain:

```
Error: Validation failed for batch 7 (150 records)

Caused by:
    0: Record 42 has invalid email
    1: missing '@' in email address
```

### The Golden Rule

> **Every `?` should either be inside a function whose name provides context,
> or should have `.context()` / `.map_err()` adding context.**

---

## Pattern: Structured Error Logging

Printing errors to stderr with `eprintln!` works for small programs. For production
applications, you need structured logging with severity levels and error chains.

### Using the `tracing` Crate

```rust
use tracing::{error, warn, info, debug};

fn handle_request(req: Request) -> Result<Response, AppError> {
    let user = match authenticate(&req) {
        Ok(user) => user,
        Err(e) => {
            warn!(
                error = %e,
                path = %req.path(),
                method = %req.method(),
                "Authentication failed"
            );
            return Ok(Response::unauthorized());
        }
    };

    let result = process(&req, &user);

    match result {
        Ok(data) => {
            info!(
                user_id = %user.id,
                path = %req.path(),
                "Request processed successfully"
            );
            Ok(Response::ok(data))
        }
        Err(e) => {
            error!(
                error = ?e,           // Debug format — includes full chain
                user_id = %user.id,
                path = %req.path(),
                "Request processing failed"
            );
            Ok(Response::internal_error())
        }
    }
}
```

### Log Levels for Errors

| Level | When to Use | Example |
|-------|------------|---------|
| `error!` | Unrecoverable, needs human attention | Database connection lost |
| `warn!` | Degraded but operational, or suspicious | Cache miss, slow query, auth failure |
| `info!` | Expected events worth recording | User login, config loaded |
| `debug!` | Detailed info for debugging | Retry attempt #3, parsed config values |

### Logging Error Chains

```rust
// With anyhow, use {:#} for the full chain on one line:
error!("Operation failed: {:#}", err);
// Output: "Operation failed: read config: open file: No such file or directory"

// Use {:?} for the multi-line Debug format:
error!("Operation failed: {:?}", err);
// Output:
//   Operation failed: read config
//
//   Caused by:
//       0: open file
//       1: No such file or directory (os error 2)
```

### Printing the Entire Error Chain Manually

If you're not using `anyhow`, you can walk the chain with `source()`:

```rust
use std::error::Error;

fn format_error_chain(err: &dyn Error) -> String {
    let mut chain = String::new();
    chain.push_str(&err.to_string());

    let mut current = err.source();
    while let Some(cause) = current {
        chain.push_str(&format!("\n  Caused by: {}", cause));
        current = cause.source();
    }

    chain
}
```

---

## Pattern: Graceful Degradation

Not every error should crash your program. Many errors can be handled by falling
back to a default, retrying, or skipping the failed operation.

### Using Default Values

```rust
use std::time::Duration;

struct AppConfig {
    timeout: Duration,
    max_retries: u32,
    cache_dir: String,
}

fn load_config_gracefully() -> AppConfig {
    // Try to load from file; if anything fails, use sensible defaults.
    let raw = std::fs::read_to_string("config.toml").ok();

    let parsed: Option<TomlConfig> = raw
        .as_deref()
        .and_then(|s| toml::from_str(s).ok());

    AppConfig {
        timeout: parsed
            .as_ref()
            .and_then(|c| c.timeout_secs)
            .map(Duration::from_secs)
            .unwrap_or(Duration::from_secs(30)),

        max_retries: parsed
            .as_ref()
            .and_then(|c| c.max_retries)
            .unwrap_or(3),

        cache_dir: parsed
            .as_ref()
            .and_then(|c| c.cache_dir.clone())
            .unwrap_or_else(|| "/tmp/myapp".to_string()),
    }
}
```

### Retry with Exponential Backoff

```rust
use std::time::Duration;
use std::thread;

fn retry_with_backoff<T, E, F>(mut operation: F, max_retries: u32) -> Result<T, E>
where
    F: FnMut() -> Result<T, E>,
    E: std::fmt::Display,
{
    let mut attempt = 0;
    loop {
        match operation() {
            Ok(value) => return Ok(value),
            Err(e) => {
                attempt += 1;
                if attempt >= max_retries {
                    eprintln!("Final attempt {} failed: {}", attempt, e);
                    return Err(e);
                }

                let backoff = Duration::from_millis(100 * 2u64.pow(attempt));
                eprintln!(
                    "Attempt {} failed: {}. Retrying in {:?}...",
                    attempt, e, backoff
                );
                thread::sleep(backoff);
            }
        }
    }
}

// Usage:
fn fetch_data() -> Result<String, reqwest::Error> {
    // ... make HTTP request
    # Ok(String::new())
}

fn main() {
    let data = retry_with_backoff(fetch_data, 5);
}
```

### Circuit Breaker Pattern

When an external service is down, hammering it with retries makes things worse.
A circuit breaker stops trying after a threshold of failures:

```rust
use std::time::{Duration, Instant};

struct CircuitBreaker {
    failure_count: u32,
    failure_threshold: u32,
    last_failure: Option<Instant>,
    cooldown: Duration,
}

#[derive(Debug)]
enum CircuitState {
    Closed,     // Normal operation — requests pass through
    Open,       // Too many failures — requests are rejected immediately
    HalfOpen,   // Cooldown elapsed — allow one test request
}

impl CircuitBreaker {
    fn new(threshold: u32, cooldown: Duration) -> Self {
        Self {
            failure_count: 0,
            failure_threshold: threshold,
            last_failure: None,
            cooldown,
        }
    }

    fn state(&self) -> CircuitState {
        if self.failure_count >= self.failure_threshold {
            if let Some(last) = self.last_failure {
                if last.elapsed() >= self.cooldown {
                    return CircuitState::HalfOpen;
                }
            }
            CircuitState::Open
        } else {
            CircuitState::Closed
        }
    }

    fn call<T, E, F>(&mut self, operation: F) -> Result<T, CircuitError<E>>
    where
        F: FnOnce() -> Result<T, E>,
    {
        match self.state() {
            CircuitState::Open => Err(CircuitError::CircuitOpen),
            CircuitState::Closed | CircuitState::HalfOpen => {
                match operation() {
                    Ok(val) => {
                        self.failure_count = 0; // Reset on success
                        Ok(val)
                    }
                    Err(e) => {
                        self.failure_count += 1;
                        self.last_failure = Some(Instant::now());
                        Err(CircuitError::Inner(e))
                    }
                }
            }
        }
    }
}

#[derive(Debug)]
enum CircuitError<E> {
    CircuitOpen,
    Inner(E),
}
```

### When to Degrade vs. When to Fail

| Scenario | Strategy |
|----------|----------|
| Config file missing | Use defaults, log a warning |
| Cache unavailable | Bypass cache, hit the source directly |
| Non-critical API down | Return partial data, note what's missing |
| Database unreachable | **Fail** — this is usually fatal |
| Invalid user input | Return a 400 / validation error |
| Corrupted data file | **Fail** — don't silently produce wrong results |

---

## Anti-Patterns to Avoid

### 1. `.unwrap()` Everywhere

```rust
// 🚫 BAD — mystery panic in production
let config = fs::read_to_string("config.toml").unwrap();
let parsed: Config = toml::from_str(&config).unwrap();

// ✅ BETTER — at least say WHY you expect this to succeed
let config = fs::read_to_string("config.toml")
    .expect("config.toml must exist in the working directory");

// ✅ BEST — propagate the error
let config = fs::read_to_string("config.toml")?;
```

### 2. Swallowing Errors Silently

```rust
// 🚫 BAD — the underscore throws away the error entirely
if let Err(_) = save_to_database(&record) {
    // silently lost forever
}

// 🚫 ALSO BAD — .ok() discards errors
let _ = send_notification(user_id).ok();

// ✅ BETTER — at least log it
if let Err(e) = save_to_database(&record) {
    eprintln!("Warning: failed to save record: {}", e);
}

// ✅ BEST — handle or propagate
save_to_database(&record)?;
```

### 3. String Errors

```rust
// 🚫 BAD — not composable, can't match on variants, loses type info
fn parse_input(s: &str) -> Result<Data, String> {
    let val: i32 = s.parse().map_err(|e| format!("parse error: {}", e))?;
    Ok(Data(val))
}

// ✅ GOOD — typed, composable, implements Error trait
#[derive(Debug, thiserror::Error)]
enum ParseInputError {
    #[error("Failed to parse integer: {0}")]
    InvalidNumber(#[from] std::num::ParseIntError),
}

fn parse_input(s: &str) -> Result<Data, ParseInputError> {
    let val: i32 = s.parse()?;
    Ok(Data(val))
}
```

### 4. Panicking in Library Code

```rust
// 🚫 BAD — library function that panics
pub fn divide(a: f64, b: f64) -> f64 {
    if b == 0.0 {
        panic!("Division by zero!"); // Caller can't recover!
    }
    a / b
}

// ✅ GOOD — let the caller decide
pub fn divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 {
        return Err(MathError::DivisionByZero);
    }
    Ok(a / b)
}
```

### 5. Ignoring `#[must_use]` Warnings

`Result` is marked `#[must_use]`. If you call a function that returns `Result`
and don't use the return value, the compiler warns you. **Never ignore this.**

```rust
// 🚫 BAD — compiler warns, and you're ignoring a potential error
write_to_log("server started");

// ✅ GOOD — explicitly acknowledge you don't care
let _ = write_to_log("server started");

// ✅ BETTER — actually handle it
if let Err(e) = write_to_log("server started") {
    eprintln!("Logging failed: {}", e);
}
```

### 6. Overusing `Box<dyn Error>` in Libraries

```rust
// 🚫 BAD in a library — caller can't inspect the error
pub fn connect(url: &str) -> Result<Connection, Box<dyn std::error::Error>> { ... }

// ✅ GOOD — specific error type
pub fn connect(url: &str) -> Result<Connection, ConnectionError> { ... }
```

---

## Error Handling in Different Contexts

### CLI Applications

CLI tools should print a human-readable error chain and exit with a proper status code.

```rust
use anyhow::{Context, Result};
use std::process::ExitCode;

fn run() -> Result<()> {
    let args: Vec<String> = std::env::args().collect();
    let path = args.get(1)
        .context("Usage: myapp <config-file>")?;

    let config = load_config(path)
        .with_context(|| format!("Failed to load config from '{}'", path))?;

    process(&config)?;

    Ok(())
}

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            // Print the full error chain
            eprintln!("Error: {:#}", err);
            ExitCode::FAILURE
        }
    }
}
```

Output:

```
Error: Failed to load config from 'prod.toml': Config file not found: prod.toml: No such file or directory (os error 2)
```

### Web Servers

Convert domain errors into appropriate HTTP status codes:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ApiError {
    #[error("Resource not found: {0}")]
    NotFound(String),

    #[error("Invalid request: {0}")]
    BadRequest(String),

    #[error("Unauthorized")]
    Unauthorized,

    #[error("Internal error")]
    Internal(#[source] anyhow::Error),
}

// In an Actix-web or Axum handler, you'd implement a conversion:
impl ApiError {
    fn status_code(&self) -> u16 {
        match self {
            ApiError::NotFound(_) => 404,
            ApiError::BadRequest(_) => 400,
            ApiError::Unauthorized => 401,
            ApiError::Internal(_) => 500,
        }
    }
}

// In the handler:
fn get_user(user_id: u64) -> Result<User, ApiError> {
    let user = db::find_user(user_id)
        .map_err(|e| ApiError::Internal(e.into()))?
        .ok_or_else(|| ApiError::NotFound(format!("User {}", user_id)))?;

    Ok(user)
}
```

**Important:** Never expose internal error details to the client in production.
Log the full error server-side; return a sanitized message to the client.

### Libraries

Libraries have the strictest error handling requirements:

```rust
//  DO:
// ✅ Define a public error enum with thiserror
// ✅ Implement std::error::Error (thiserror does this)
// ✅ Make errors Send + Sync (they are by default if fields are)
// ✅ Document which functions return which errors
// ✅ Provide From impls for common conversions

// DON'T:
// 🚫 panic! or unwrap() on recoverable errors
// 🚫 Use anyhow in your public API
// 🚫 Print to stderr — let the caller decide how to present errors
// 🚫 Use String as an error type
```

### Embedded / `no_std` Environments

Without the standard library, you can't use `Box`, `String`, or heap allocation:

```rust
#![no_std]

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum SensorError {
    Timeout,
    ChecksumMismatch,
    OutOfRange,
    BusError(u8), // Error code from the hardware bus
}

// No String, no Box, no allocation — just plain enums and fixed data.
fn read_temperature(sensor_id: u8) -> Result<i16, SensorError> {
    let raw = bus_read(sensor_id).map_err(SensorError::BusError)?;

    if raw == 0xFFFF {
        return Err(SensorError::Timeout);
    }

    let temp = convert_raw_to_celsius(raw);
    if temp < -40 || temp > 125 {
        return Err(SensorError::OutOfRange);
    }

    Ok(temp)
}
```

---

## Testing Error Paths

Good tests don't just verify the happy path — they verify that errors are correct too.

### Testing Panics with `#[should_panic]`

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn test_invalid_index_panics() {
        let v = vec![1, 2, 3];
        let _ = v[10]; // panics
    }

    // More precise: check the exact panic message
    #[test]
    #[should_panic(expected = "Division by zero")]
    fn test_divide_by_zero_panics() {
        divide_or_panic(10, 0);
    }
}
```

### Testing `Result` Values

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_valid_input() {
        let result = parse_age("25");
        assert!(result.is_ok());
        assert_eq!(result.unwrap(), 25);
    }

    #[test]
    fn test_parse_negative_fails() {
        let result = parse_age("-5");
        assert!(result.is_err());
    }

    // Pattern match to verify the EXACT error variant
    #[test]
    fn test_missing_config_gives_file_not_found() {
        let result = load_config("/nonexistent/path.toml");
        match result {
            Err(ConfigError::FileNotFound { path, .. }) => {
                assert_eq!(path, "/nonexistent/path.toml");
            }
            other => panic!("Expected FileNotFound, got {:?}", other),
        }
    }
}
```

### Testing Error Messages

```rust
#[test]
fn test_error_display_message() {
    let err = ConfigError::MissingField {
        field: "database_url".to_string(),
    };
    assert_eq!(err.to_string(), "Missing required field: database_url");
}

#[test]
fn test_error_message_contains_context() {
    let result = parse_port("not_a_number");
    let err = result.unwrap_err();
    // Don't check exact string — it might change. Check it contains key info:
    let msg = err.to_string();
    assert!(msg.contains("not_a_number"), "Error should mention the input");
    assert!(msg.contains("port"), "Error should mention what was being parsed");
}
```

### Testing Error Chains (Source)

```rust
use std::error::Error;

#[test]
fn test_config_error_wraps_io_error() {
    let result = load_config("/nonexistent/file.toml");
    let err = result.unwrap_err();

    // Verify the error has a source (the underlying io::Error)
    assert!(err.source().is_some(), "ConfigError should wrap an io::Error");

    // Verify the source is specifically an io::Error
    let source = err.source().unwrap();
    assert!(
        source.downcast_ref::<std::io::Error>().is_some(),
        "Source should be an io::Error"
    );
}
```

### Using `assert_matches!` (Nightly or with a Crate)

```rust
// With the `assert_matches` crate or on nightly:
use assert_matches::assert_matches;

#[test]
fn test_specific_variant() {
    let err = process_record("bad data").unwrap_err();
    assert_matches!(err, ProcessError::Validation { line: 1, .. });
}
```

---

## Real-World Case Study: A File Processor

Let's tie everything together with a realistic example: a program that reads
a CSV of user records, validates each row, and writes a cleaned output file.

```rust
use std::fs::{self, File};
use std::io::{BufRead, BufReader, Write};
use std::path::Path;
use thiserror::Error;

// ── Error types ──────────────────────────────────────────────

#[derive(Debug, Error)]
pub enum ProcessorError {
    #[error("Failed to open input file '{path}'")]
    OpenInput {
        path: String,
        #[source]
        source: std::io::Error,
    },

    #[error("Failed to create output file '{path}'")]
    CreateOutput {
        path: String,
        #[source]
        source: std::io::Error,
    },

    #[error("Failed to read line {line_number} from '{path}'")]
    ReadLine {
        path: String,
        line_number: usize,
        #[source]
        source: std::io::Error,
    },

    #[error("Validation error on line {line_number}: {message}")]
    Validation {
        line_number: usize,
        message: String,
    },

    #[error("Failed to write output")]
    Write(#[source] std::io::Error),
}

// ── Data types ───────────────────────────────────────────────

#[derive(Debug)]
struct UserRecord {
    name: String,
    email: String,
    age: u8,
}

// ── Core logic ───────────────────────────────────────────────

fn parse_record(line: &str, line_number: usize) -> Result<UserRecord, ProcessorError> {
    let fields: Vec<&str> = line.split(',').collect();

    if fields.len() != 3 {
        return Err(ProcessorError::Validation {
            line_number,
            message: format!("Expected 3 fields, found {}", fields.len()),
        });
    }

    let name = fields[0].trim().to_string();
    if name.is_empty() {
        return Err(ProcessorError::Validation {
            line_number,
            message: "Name cannot be empty".to_string(),
        });
    }

    let email = fields[1].trim().to_string();
    if !email.contains('@') {
        return Err(ProcessorError::Validation {
            line_number,
            message: format!("Invalid email: '{}'", email),
        });
    }

    let age: u8 = fields[2].trim().parse().map_err(|_| ProcessorError::Validation {
        line_number,
        message: format!("Invalid age: '{}'", fields[2].trim()),
    })?;

    Ok(UserRecord { name, email, age })
}

// ── The processor ────────────────────────────────────────────

pub fn process_csv(input_path: &str, output_path: &str) -> Result<ProcessResult, ProcessorError> {
    // Open input
    let file = File::open(input_path).map_err(|e| ProcessorError::OpenInput {
        path: input_path.to_string(),
        source: e,
    })?;
    let reader = BufReader::new(file);

    // Create output
    let mut output = File::create(output_path).map_err(|e| ProcessorError::CreateOutput {
        path: output_path.to_string(),
        source: e,
    })?;

    let mut processed = 0;
    let mut skipped = 0;
    let mut errors: Vec<String> = Vec::new();

    for (index, line_result) in reader.lines().enumerate() {
        let line_number = index + 1;

        let line = line_result.map_err(|e| ProcessorError::ReadLine {
            path: input_path.to_string(),
            line_number,
            source: e,
        })?;

        // Skip header
        if line_number == 1 && line.starts_with("name") {
            continue;
        }

        // Skip blank lines
        if line.trim().is_empty() {
            skipped += 1;
            continue;
        }

        // Parse and validate — collect errors instead of failing on first
        match parse_record(&line, line_number) {
            Ok(record) => {
                writeln!(output, "{},{},{}", record.name, record.email, record.age)
                    .map_err(ProcessorError::Write)?;
                processed += 1;
            }
            Err(e) => {
                // Graceful degradation: log the error, skip the line, continue
                errors.push(format!("Line {}: {}", line_number, e));
                skipped += 1;
            }
        }
    }

    Ok(ProcessResult {
        processed,
        skipped,
        errors,
    })
}

pub struct ProcessResult {
    pub processed: usize,
    pub skipped: usize,
    pub errors: Vec<String>,
}

// ── Main ─────────────────────────────────────────────────────

fn main() {
    match process_csv("users.csv", "cleaned_users.csv") {
        Ok(result) => {
            println!("Done! Processed: {}, Skipped: {}", result.processed, result.skipped);
            if !result.errors.is_empty() {
                eprintln!("\nWarnings:");
                for err in &result.errors {
                    eprintln!("  {}", err);
                }
            }
        }
        Err(e) => {
            eprintln!("Fatal error: {}", e);
            if let Some(source) = std::error::Error::source(&e) {
                eprintln!("  Caused by: {}", source);
            }
            std::process::exit(1);
        }
    }
}
```

**What this example demonstrates:**

- **Custom error types** with `thiserror` — each variant carries context
- **Error conversion** at boundaries — `io::Error` → `ProcessorError::OpenInput`
- **Context enrichment** — every error knows the file path and line number
- **Graceful degradation** — bad records are skipped, not fatal
- **Error chain preservation** — `#[source]` keeps the underlying cause
- **Structured reporting** — summary at the end with all warnings

---

## The Rust Community's Error Handling Guidelines

### RFC 236: Error Conventions

[RFC 236](https://rust-lang.github.io/rfcs/0236-error-conventions.html) established
the foundational conventions for error handling in Rust:

- **Module-level error types**: Each module should define its own error type.
- **The `Error` trait**: All error types should implement `std::error::Error`.
- **`From` conversions**: Use `From<OtherError>` to enable `?` propagation.
- **Descriptive messages**: `Display` should produce a human-readable message, lowercase, without trailing punctuation.

### The Rust API Guidelines

The [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) add more detail:

| Guideline | Rule |
|-----------|------|
| **C-GOOD-ERR** | Error types should be meaningful, not just `String` or `()` |
| **C-SEND-SYNC** | Error types should be `Send + Sync` for use across threads |
| **C-MEANINGFUL** | Error messages should include context about what went wrong |
| **C-CONV** | Provide `From` impls so `?` works smoothly |

### Display Format Convention

```rust
// ✅ GOOD: lowercase, no trailing period, brief
#[error("invalid header: expected {expected}, found {found}")]

// 🚫 BAD: starts uppercase, ends with period
#[error("Invalid header: expected {expected}, found {found}.")]
```

This convention exists because errors are often embedded in larger messages:

```
"failed to parse config: invalid header: expected 3, found 5"
//                       ^ lowercase flows naturally in a sentence
```

### When the Ecosystem Recommends What

| Situation | Community Consensus |
|-----------|-------------------|
| New library crate | `thiserror` for public error types |
| New application | `anyhow` in binary, `thiserror` in any internal libraries |
| Quick prototype | `anyhow` everywhere, or even `Box<dyn Error>` |
| `no_std` crate | Manual `Error` impl (no proc-macro deps), or plain enums |
| Performance-critical code | Custom enums, no heap allocation in error path |
| Async code | Ensure errors are `Send + Sync + 'static` |

### The Evolution of Error Handling in Rust

```
2015:  try!() macro, custom Error impls everywhere
2016:  error-chain crate (now deprecated)
2018:  failure crate (now deprecated)
2019:  ? operator stabilized, anyhow + thiserror released
2021:  Community converges on anyhow + thiserror as standard
2024+: std::error::Error in core (no_std support improving)
```

The ecosystem has settled. **`thiserror` for libraries, `anyhow` for applications**
is the standard advice in 2024 and beyond.

---

## Exercises

### Exercise 1: Error Conversion Layer

Build a `UserService` module that wraps a file-based "database":

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum UserError {
    // Define variants for: user not found, duplicate email,
    // storage error (wrapping io::Error), and validation error.
    // Each variant should carry appropriate context.
}

pub struct UserService {
    data_dir: String,
}

impl UserService {
    /// Load a user by ID. File is at `{data_dir}/{id}.json`.
    pub fn get_user(&self, id: u64) -> Result<User, UserError> {
        todo!()
    }

    /// Save a user. Validate email contains '@', name is non-empty.
    pub fn save_user(&self, user: &User) -> Result<(), UserError> {
        todo!()
    }
}
```

**Requirements:**
- Convert `io::Error` and `serde_json::Error` into `UserError` variants
- Add context (user ID, file path) to every error
- Write tests for both happy path and each error variant

### Exercise 2: Retry Logic

Implement a generic `with_retry` function:

```rust
/// Retry an operation up to `max_attempts` times.
/// Only retry if `should_retry` returns true for the error.
/// Use exponential backoff between retries.
fn with_retry<T, E, F, R>(
    mut operation: F,
    should_retry: R,
    max_attempts: u32,
) -> Result<T, E>
where
    F: FnMut() -> Result<T, E>,
    R: Fn(&E) -> bool,
    E: std::fmt::Display,
{
    todo!()
}
```

**Test it with:**
- An operation that fails 3 times then succeeds
- An operation that fails with a non-retryable error (should stop immediately)
- An operation that exhausts all retries

### Exercise 3: Error Report Formatter

Write a function that takes any `&dyn std::error::Error` and produces a
nicely formatted, numbered error chain:

```
Error: Failed to process batch 42
  1. Failed to read input file
  2. config.toml: permission denied (os error 13)
```

```rust
fn format_error_report(err: &dyn std::error::Error) -> String {
    todo!()
}
```

**Test it with nested errors (3+ levels deep) and ensure the numbering is correct.**

---

## Summary

| Principle | Rule |
|-----------|------|
| **Bugs vs. failures** | `panic!` for bugs, `Result` for expected failures |
| **Library vs. app** | `thiserror` for libraries, `anyhow` for applications |
| **Context** | Every error should answer "what were we trying to do?" |
| **Conversion** | Convert library errors to domain errors at module boundaries |
| **Degradation** | Not every error is fatal — use defaults, retries, fallbacks |
| **Testing** | Test error paths as thoroughly as happy paths |
| **Display format** | Lowercase, no trailing punctuation, brief and composable |
| **Don't panic in libs** | Return `Result` — let the caller decide |
| **Don't swallow errors** | At minimum, log them; ideally, propagate them |
| **Chains** | Preserve `source()` so the full cause is available |

You've now completed the entire Error Handling stage. You know how to use `Result`
and `Option`, the `?` operator, custom error types, `thiserror`, `anyhow`, and
the real-world patterns that tie them all together. Error handling in Rust isn't
just about preventing crashes — it's about building software that **communicates
clearly when something goes wrong** and gives the caller the power to decide
what to do about it.

---

**Previous:** [← thiserror & anyhow](./06-thiserror-and-anyhow.md)

**Next Stage:** [Stage 8: Generics & Traits →](../08-generics-and-traits/README.md)

---

<p align="center"><i>Tutorial 7 of 7 — Stage 7: Error Handling</i></p>

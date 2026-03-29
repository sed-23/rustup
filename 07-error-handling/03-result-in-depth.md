# Result\<T, E\> in Depth 🎯

> **"Result is Rust's way of saying: every operation that can fail _must_ tell you how it can fail — and the compiler enforces it."**

---

## Table of Contents

- [Result Refresher](#result-refresher)
- [Basic Pattern Matching on Result](#basic-pattern-matching-on-result)
- [The Combinators — Result's Power Methods](#the-combinators--results-power-methods)
- [unwrap, expect, and When They're OK](#unwrap-expect-and-when-theyre-ok)
- [Result in the Standard Library](#result-in-the-standard-library)
- [Converting Between Option and Result](#converting-between-option-and-result)
- [Collecting Results](#collecting-results)
- [Result vs Other Languages](#result-vs-other-languages)
- [Deep Dive: How Result Is Zero-Cost](#deep-dive-how-result-is-zero-cost)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Result Refresher

At its core, `Result` is just an enum with two variants:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

That's it. No magic, no runtime overhead, no hidden control flow. `T` is the success type, `E` is the error type. Every function that can fail returns one of these two variants, and the caller **must** handle both possibilities — the compiler won't let you forget.

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    let result = divide(10.0, 3.0);
    println!("{:?}", result); // Ok(3.3333333333333335)

    let result = divide(10.0, 0.0);
    println!("{:?}", result); // Err("division by zero")
}
```

Because `Result` is an enum, everything you know about enums applies: pattern matching, destructuring, method calls. There is no special syntax — just Rust's type system doing its job.

### Result Is in the Prelude

You never need to write `std::result::Result` — it's imported into every Rust file automatically via the prelude, along with its variants `Ok` and `Err`:

```rust
// You can write this directly — no imports needed:
let x: Result<i32, String> = Ok(42);
let y: Result<i32, String> = Err("oops".to_string());
```

---

## Basic Pattern Matching on Result

### The Classic `match`

The most explicit way to handle a `Result`:

```rust
use std::fs;

fn main() {
    let content = fs::read_to_string("hello.txt");

    match content {
        Ok(text) => println!("File contents: {text}"),
        Err(e) => println!("Failed to read file: {e}"),
    }
}
```

The compiler guarantees you handle both arms. If you remove the `Err` arm, this won't compile.

### `if let` — When You Only Care About One Variant

Sometimes you only want to act on success (or only on failure):

```rust
use std::fs;

fn main() {
    // Only care about success
    if let Ok(text) = fs::read_to_string("config.toml") {
        println!("Config loaded: {text}");
    }

    // Only care about failure
    if let Err(e) = fs::remove_file("temp.txt") {
        eprintln!("Cleanup failed: {e}");
    }
}
```

### `let ... else` — Early Return on Error

Rust 1.65+ gives us `let ... else` for the common "unwrap or bail out" pattern:

```rust
use std::fs;

fn load_config() -> Result<String, std::io::Error> {
    let Ok(text) = fs::read_to_string("config.toml") else {
        return Err(std::io::Error::new(
            std::io::ErrorKind::NotFound,
            "config missing",
        ));
    };

    // `text` is a plain String here — no Result wrapper
    Ok(text.to_uppercase())
}
```

### Nested Matches

When one fallible operation depends on another, matches nest naturally:

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username() -> Result<String, io::Error> {
    match File::open("username.txt") {
        Ok(mut file) => {
            let mut name = String::new();
            match file.read_to_string(&mut name) {
                Ok(_) => Ok(name.trim().to_string()),
                Err(e) => Err(e),
            }
        }
        Err(e) => Err(e),
    }
}
```

This works but gets deeply indented fast. That's exactly why combinators and the `?` operator exist — we'll cover `?` in the next chapter, but first let's master the combinators.

---

## The Combinators — Result's Power Methods

Combinators are methods on `Result` that let you transform, chain, and unwrap results without explicit `match` blocks. Think of them as a pipeline for fallible values.

### `map()` — Transform the Ok Value

`map()` applies a function to the `Ok` value and leaves `Err` untouched:

```rust
fn main() {
    let result: Result<i32, String> = Ok(5);

    let doubled = result.map(|v| v * 2);
    assert_eq!(doubled, Ok(10));

    // If it's an Err, map does nothing:
    let err_result: Result<i32, String> = Err("bad".into());
    let still_err = err_result.map(|v| v * 2);
    assert_eq!(still_err, Err("bad".into()));
}
```

**Real-world use** — parsing a string into an integer, then doubling:

```rust
fn doubled_parse(s: &str) -> Result<i32, std::num::ParseIntError> {
    s.parse::<i32>().map(|n| n * 2)
}

fn main() {
    assert_eq!(doubled_parse("21"), Ok(42));
    assert!(doubled_parse("abc").is_err());
}
```

### `map_err()` — Transform the Err Value

The mirror of `map()`. Leaves `Ok` untouched, transforms `Err`:

```rust
use std::fs;

fn read_file(path: &str) -> Result<String, String> {
    fs::read_to_string(path)
        .map_err(|e| format!("failed to read '{path}': {e}"))
}

fn main() {
    match read_file("missing.txt") {
        Ok(text) => println!("{text}"),
        Err(msg) => eprintln!("{msg}"),
        // "failed to read 'missing.txt': No such file or directory"
    }
}
```

This is incredibly common for converting between error types — you'll use `map_err()` constantly in real codebases.

### `and_then()` — Chain Fallible Operations (Flatmap)

`and_then()` is the big one. When your transformation itself can fail, `map()` would give you `Result<Result<T, E>, E>` — nested results. `and_then()` flattens that:

```rust
fn parse_and_double(s: &str) -> Result<i32, String> {
    s.parse::<i32>()
        .map_err(|e| e.to_string())
        .and_then(|n| {
            if n > 1_000_000 {
                Err("number too large".to_string())
            } else {
                Ok(n * 2)
            }
        })
}

fn main() {
    assert_eq!(parse_and_double("21"), Ok(42));
    assert_eq!(parse_and_double("abc"), Err("invalid digit found in string".into()));
    assert_eq!(parse_and_double("9999999"), Err("number too large".into()));
}
```

**Chaining multiple `and_then()` calls** builds a pipeline of fallible steps:

```rust
fn pipeline(input: &str) -> Result<String, String> {
    input
        .parse::<i32>()
        .map_err(|e| format!("parse error: {e}"))
        .and_then(|n| {
            if n >= 0 { Ok(n) } else { Err("negative number".into()) }
        })
        .and_then(|n| {
            let sqrt = (n as f64).sqrt();
            if sqrt.fract() == 0.0 {
                Ok(sqrt as i32)
            } else {
                Err(format!("{n} is not a perfect square"))
            }
        })
        .map(|n| format!("√ = {n}"))
}

fn main() {
    println!("{:?}", pipeline("49"));   // Ok("√ = 7")
    println!("{:?}", pipeline("-4"));   // Err("negative number")
    println!("{:?}", pipeline("50"));   // Err("50 is not a perfect square")
    println!("{:?}", pipeline("abc"));  // Err("parse error: invalid digit ...")
}
```

### `or_else()` — Provide a Fallback That Also Returns Result

`or_else()` is called only when the result is `Err`. It lets you try a recovery strategy:

```rust
use std::fs;

fn read_config() -> Result<String, String> {
    fs::read_to_string("config.toml")
        .map_err(|e| e.to_string())
        .or_else(|_| {
            // Primary config missing? Try the fallback location.
            fs::read_to_string("/etc/myapp/config.toml")
                .map_err(|e| format!("no config found anywhere: {e}"))
        })
}
```

### `unwrap_or()` — Default Value on Error

Returns the `Ok` value, or a provided default if `Err`:

```rust
fn main() {
    let port: u16 = std::env::var("PORT")
        .ok()                          // Result -> Option
        .and_then(|s| s.parse().ok())  // Option<u16>
        .unwrap_or(8080);              // default port

    // Or directly with Result:
    let count: i32 = "not_a_number".parse::<i32>().unwrap_or(0);
    assert_eq!(count, 0);
}
```

### `unwrap_or_else()` — Compute Default Lazily

Like `unwrap_or()`, but the default is computed by a closure — only called if needed:

```rust
fn expensive_default() -> i32 {
    println!("computing expensive default...");
    42
}

fn main() {
    let ok_result: Result<i32, &str> = Ok(10);
    let val = ok_result.unwrap_or_else(|_| expensive_default());
    // "computing expensive default..." is NOT printed — closure never runs
    assert_eq!(val, 10);

    let err_result: Result<i32, &str> = Err("oops");
    let val = err_result.unwrap_or_else(|_| expensive_default());
    // "computing expensive default..." IS printed
    assert_eq!(val, 42);
}
```

### `unwrap_or_default()` — Use the Default Trait

If `T` implements `Default`, you can use this for the most concise fallback:

```rust
fn main() {
    let result: Result<i32, &str> = Err("error");
    assert_eq!(result.unwrap_or_default(), 0);   // i32::default() == 0

    let result: Result<String, &str> = Err("error");
    assert_eq!(result.unwrap_or_default(), "");   // String::default() == ""

    let result: Result<Vec<i32>, &str> = Err("error");
    assert_eq!(result.unwrap_or_default(), vec![]); // Vec::default() == []
}
```

### Combinator Cheat Sheet

| Combinator           | Acts on | Transforms to       | Short description                      |
| -------------------- | ------- | -------------------- | -------------------------------------- |
| `map(f)`             | `Ok(t)` | `Ok(f(t))`          | Transform success value                |
| `map_err(f)`         | `Err(e)`| `Err(f(e))`         | Transform error value                  |
| `and_then(f)`        | `Ok(t)` | `f(t)` → `Result`   | Chain fallible operations              |
| `or_else(f)`         | `Err(e)`| `f(e)` → `Result`   | Fallback on error                      |
| `unwrap_or(val)`     | `Err`   | `val`                | Static default                         |
| `unwrap_or_else(f)`  | `Err(e)`| `f(e)`               | Computed default                       |
| `unwrap_or_default()`| `Err`   | `T::default()`       | Default trait                          |
| `ok()`               | Both    | `Option<T>`          | Discard error info                     |
| `err()`              | Both    | `Option<E>`          | Discard success info                   |
| `is_ok()`            | Both    | `bool`               | Check if Ok                            |
| `is_err()`           | Both    | `bool`               | Check if Err                           |

---

## unwrap, expect, and When They're OK

### `unwrap()` — The Blunt Instrument

`unwrap()` extracts the `Ok` value or panics with a generic message:

```rust
fn main() {
    let val: Result<i32, &str> = Ok(42);
    assert_eq!(val.unwrap(), 42); // fine

    let err: Result<i32, &str> = Err("something broke");
    err.unwrap();
    // thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "something broke"'
}
```

The panic message is unhelpful — it tells you _what_ went wrong but not _where_ or _why_ in your logic.

### `expect()` — Panic with Context

`expect()` does the same thing but lets you provide a message explaining what you were trying to do:

```rust
use std::fs::File;

fn main() {
    let file = File::open("critical.dat")
        .expect("critical.dat must exist in the working directory");
    // thread 'main' panicked at 'critical.dat must exist in the working directory: 
    //   Os { code: 2, kind: NotFound, message: "No such file or directory" }'
}
```

**Always prefer `expect()` over `unwrap()`.** The `expect` message should describe the _invariant_ that was violated, not the error itself.

```rust
// BAD — just repeats the error
let x = val.expect("parse failed");

// GOOD — explains what you expected to be true
let x = val.expect("PORT env var should be a valid u16");
```

### When Is `unwrap()`/`expect()` Acceptable?

| Context              | Verdict  | Why                                                        |
| -------------------- | -------- | ---------------------------------------------------------- |
| **Tests**            | ✅ Fine  | A panic _is_ a test failure — that's what you want          |
| **Prototyping**      | ✅ Fine  | You'll replace it once the logic is fleshed out             |
| **After validation** | ✅ Fine  | You've already proven the value is `Ok`                     |
| **Examples / docs**  | ✅ Fine  | Keeps examples focused on the topic at hand                 |
| **Production logic** | ❌ Avoid | Use `?`, combinators, or explicit `match` instead           |

```rust
// After validation — this unwrap can never panic:
fn parse_known_good(s: &str) -> i32 {
    // We validated `s` upstream and know it's digits only
    debug_assert!(s.chars().all(|c| c.is_ascii_digit()));
    s.parse::<i32>().expect("pre-validated string should parse")
}
```

### Enforcing No-Unwrap with Clippy

You can ban `unwrap()` in your codebase:

```rust
// At the crate root (lib.rs or main.rs):
#![deny(clippy::unwrap_used)]
#![deny(clippy::expect_used)]  // if you want to be extra strict
```

Or in `clippy.toml` / `Cargo.toml`:

```toml
# Cargo.toml
[lints.clippy]
unwrap_used = "deny"
```

Now any `unwrap()` call is a compile error under `cargo clippy`. Teams often allow `expect()` but deny `unwrap()` — the middle ground.

---

## Result in the Standard Library

`Result` is everywhere in Rust's standard library. Here are the most common places you'll encounter it.

### File I/O

```rust
use std::fs::{self, File};
use std::io::{self, Read, Write, BufRead, BufReader};

fn file_io_examples() -> Result<(), io::Error> {
    // Opening a file
    let file: File = File::open("data.txt")?;

    // Reading entire file to string
    let content: String = fs::read_to_string("data.txt")?;

    // Writing to a file
    let mut file = File::create("output.txt")?;
    file.write_all(b"Hello, Rust!")?;

    // Reading line by line
    let file = File::open("data.txt")?;
    let reader = BufReader::new(file);
    for line in reader.lines() {
        let line: String = line?;  // each line() call returns Result
        println!("{line}");
    }

    // Removing a file
    fs::remove_file("output.txt")?;

    Ok(())
}
```

Every single I/O operation returns `Result<_, io::Error>`. You cannot accidentally ignore a failed read or write.

### Parsing Strings

```rust
fn parsing_examples() {
    // Parse to integer
    let n: Result<i32, _> = "42".parse();
    assert_eq!(n, Ok(42));

    let n: Result<i32, _> = "abc".parse();
    assert!(n.is_err());

    // Parse to float
    let f: Result<f64, _> = "3.14".parse();
    assert_eq!(f, Ok(3.14));

    // Parse to bool
    let b: Result<bool, _> = "true".parse();
    assert_eq!(b, Ok(true));

    // Parse to IP address
    use std::net::IpAddr;
    let ip: Result<IpAddr, _> = "127.0.0.1".parse();
    assert!(ip.is_ok());
}
```

### Networking

```rust
use std::net::{TcpStream, TcpListener};
use std::io::{self, Write, Read};

fn network_examples() -> Result<(), io::Error> {
    // Connecting to a server
    let mut stream: TcpStream = TcpStream::connect("127.0.0.1:8080")?;

    // Sending data
    stream.write_all(b"GET / HTTP/1.1\r\n\r\n")?;

    // Receiving data
    let mut buffer = [0u8; 1024];
    let bytes_read: usize = stream.read(&mut buffer)?;

    // Binding a listener
    let listener: TcpListener = TcpListener::bind("0.0.0.0:3000")?;

    // Accepting connections
    for incoming in listener.incoming() {
        let stream: TcpStream = incoming?;
        // handle connection...
    }

    Ok(())
}
```

### Environment Variables

```rust
use std::env;

fn env_examples() {
    // std::env::var returns Result<String, VarError>
    match env::var("HOME") {
        Ok(home) => println!("Home directory: {home}"),
        Err(env::VarError::NotPresent) => println!("HOME is not set"),
        Err(env::VarError::NotUnicode(_)) => println!("HOME is not valid UTF-8"),
    }

    // Common pattern: default value
    let editor = env::var("EDITOR").unwrap_or_else(|_| "vim".to_string());
    println!("Using editor: {editor}");
}
```

### Collections and Conversions

```rust
use std::collections::HashMap;

fn collection_examples() {
    // Constructing from iterator of pairs can fail if keys collide
    // (actually HashMap::from doesn't fail, but from_str does)

    // String → number conversions
    let numbers: Vec<&str> = vec!["1", "2", "three", "4"];

    // Filter only successful parses
    let parsed: Vec<i32> = numbers
        .iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();

    assert_eq!(parsed, vec![1, 2, 4]);
}
```

---

## Converting Between Option and Result

`Option` and `Result` are siblings — both represent "might not have a value." Rust provides clean conversions between them.

### Option → Result

**`ok_or(err)`** — Attach a static error to `None`:

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Alice".into()) } else { None }
}

fn get_user(id: u32) -> Result<String, String> {
    find_user(id).ok_or(format!("user {id} not found"))
}

fn main() {
    assert_eq!(get_user(1), Ok("Alice".into()));
    assert_eq!(get_user(99), Err("user 99 not found".into()));
}
```

**`ok_or_else(|| err)`** — Attach a lazily computed error:

```rust
fn get_user_lazy(id: u32) -> Result<String, String> {
    find_user(id).ok_or_else(|| {
        // This closure only runs if the Option is None
        log_missing_user(id);
        format!("user {id} not found")
    })
}

fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Alice".into()) } else { None }
}

fn log_missing_user(id: u32) {
    eprintln!("[WARN] looked up missing user {id}");
}
```

Use `ok_or_else` when constructing the error is expensive (allocations, I/O, logging).

### Result → Option

**`result.ok()`** — Discard the error, keep only the success:

```rust
fn main() {
    let r: Result<i32, &str> = Ok(42);
    assert_eq!(r.ok(), Some(42));

    let r: Result<i32, &str> = Err("fail");
    assert_eq!(r.ok(), None);  // error info is gone
}
```

**`result.err()`** — Discard the success, keep only the error:

```rust
fn main() {
    let r: Result<i32, &str> = Ok(42);
    assert_eq!(r.err(), None);

    let r: Result<i32, &str> = Err("fail");
    assert_eq!(r.err(), Some("fail"));
}
```

### `transpose()` — Swap the Nesting

This converts between `Option<Result<T, E>>` and `Result<Option<T>, E>`:

```rust
fn main() {
    // Option<Result<T, E>> → Result<Option<T>, E>
    let x: Option<Result<i32, &str>> = Some(Ok(5));
    let y: Result<Option<i32>, &str> = x.transpose();
    assert_eq!(y, Ok(Some(5)));

    let x: Option<Result<i32, &str>> = Some(Err("bad"));
    let y: Result<Option<i32>, &str> = x.transpose();
    assert_eq!(y, Err("bad"));

    let x: Option<Result<i32, &str>> = None;
    let y: Result<Option<i32>, &str> = x.transpose();
    assert_eq!(y, Ok(None));
}
```

This is especially useful when working with iterators that produce `Option<Result<...>>`.

### Conversion Cheat Sheet

| From                        | To                          | Method          |
| --------------------------- | --------------------------- | --------------- |
| `Option<T>`                 | `Result<T, E>`              | `.ok_or(e)`     |
| `Option<T>`                 | `Result<T, E>`              | `.ok_or_else(f)`|
| `Result<T, E>`              | `Option<T>`                 | `.ok()`         |
| `Result<T, E>`              | `Option<E>`                 | `.err()`        |
| `Option<Result<T, E>>`      | `Result<Option<T>, E>`      | `.transpose()`  |
| `Result<Option<T>, E>`      | `Option<Result<T, E>>`      | `.transpose()`  |

---

## Collecting Results

One of Rust's most elegant patterns is collecting an iterator of `Result`s into a single `Result` of a collection.

### The Basic Pattern

```rust
fn main() {
    let strings = vec!["1", "2", "3", "4", "5"];

    // Each parse returns Result<i32, ParseIntError>
    // collect() can turn Vec<Result<i32, E>> into Result<Vec<i32>, E>
    let numbers: Result<Vec<i32>, _> = strings
        .iter()
        .map(|s| s.parse::<i32>())
        .collect();

    assert_eq!(numbers, Ok(vec![1, 2, 3, 4, 5]));
}
```

### Short-Circuit Behavior

The key insight: **collecting stops at the first error**. It doesn't process all elements and then check for errors — it bails out immediately.

```rust
fn main() {
    let strings = vec!["1", "two", "3", "four", "5"];

    let numbers: Result<Vec<i32>, _> = strings
        .iter()
        .map(|s| {
            println!("parsing: {s}");
            s.parse::<i32>()
        })
        .collect();

    // Output:
    //   parsing: 1
    //   parsing: two
    // Stops here! "3", "four", "5" are never processed.

    assert!(numbers.is_err());
}
```

This is efficient — no wasted computation after the first failure.

### Collecting All Errors with `partition()`

Sometimes you want _all_ errors, not just the first. Use `partition()`:

```rust
fn main() {
    let strings = vec!["1", "two", "3", "four", "5"];

    let (successes, failures): (Vec<_>, Vec<_>) = strings
        .iter()
        .map(|s| s.parse::<i32>())
        .partition(Result::is_ok);

    let successes: Vec<i32> = successes.into_iter().map(Result::unwrap).collect();
    let failures: Vec<_> = failures.into_iter().map(Result::unwrap_err).collect();

    assert_eq!(successes, vec![1, 3, 5]);
    assert_eq!(failures.len(), 2);

    println!("Parsed: {successes:?}");
    println!("Failed: {failures:?}");
}
```

### Collecting into Other Types

`collect()` isn't limited to `Vec`. You can collect into `HashMap`, `BTreeSet`, `String`, etc:

```rust
use std::collections::HashMap;

fn main() {
    let pairs = vec!["name=Alice", "age=30", "city=Paris"];

    let map: Result<HashMap<&str, &str>, &str> = pairs
        .iter()
        .map(|s| {
            let mut parts = s.splitn(2, '=');
            match (parts.next(), parts.next()) {
                (Some(k), Some(v)) => Ok((k, v)),
                _ => Err("invalid key=value pair"),
            }
        })
        .collect();

    let map = map.unwrap();
    assert_eq!(map["name"], "Alice");
    assert_eq!(map["age"], "30");
}
```

### `sum()` and `product()` on Results

Iterators of `Result<T, E>` where `T: Sum` or `T: Product` can be summed/multiplied directly:

```rust
fn main() {
    let strings = vec!["1", "2", "3"];

    let total: Result<i32, _> = strings
        .iter()
        .map(|s| s.parse::<i32>())
        .sum();

    assert_eq!(total, Ok(6));

    // With an error present — short-circuits:
    let strings = vec!["1", "oops", "3"];
    let total: Result<i32, _> = strings
        .iter()
        .map(|s| s.parse::<i32>())
        .sum();

    assert!(total.is_err());
}
```

---

## Result vs Other Languages

How does Rust's `Result` compare to error handling in other languages?

### Haskell: `Either a b`

Haskell's `Either` is the algebraic ancestor of Rust's `Result`:

```haskell
-- Haskell
data Either a b = Left a | Right b

-- Convention: Left = error, Right = success
safeDivide :: Double -> Double -> Either String Double
safeDivide _ 0 = Left "division by zero"
safeDivide a b = Right (a / b)
```

Rust's `Result` is essentially `Either` with clearer naming — `Ok` instead of `Right`, `Err` instead of `Left`.

### OCaml: `result` type

OCaml added a `result` type in 4.03, mirroring the same idea:

```ocaml
(* OCaml *)
type ('a, 'b) result = Ok of 'a | Error of 'b

let safe_divide a b =
  if b = 0.0 then Error "division by zero"
  else Ok (a /. b)
```

Nearly identical to Rust. Not a coincidence — Rust drew heavy inspiration from ML-family languages.

### Swift: `Result<Success, Failure>`

Swift 5 introduced a `Result` type that closely mirrors Rust's:

```swift
// Swift
enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}

func divide(_ a: Double, by b: Double) -> Result<Double, DivisionError> {
    guard b != 0 else { return .failure(.divisionByZero) }
    return .success(a / b)
}
```

The key difference: Swift's `Failure` must conform to the `Error` protocol. Rust's `E` can be anything.

### Go: Multiple Return Values

Go uses a convention of returning `(value, error)` tuples:

```go
// Go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// The caller:
result, err := divide(10, 0)
if err != nil {
    log.Fatal(err)
}
```

**Problems with Go's approach:**
- Nothing stops you from ignoring `err`
- Nothing stops you from using `result` when `err != nil`
- `err` is `interface{}` — you lose type safety
- The compiler **does not enforce** error checking

### TypeScript / JavaScript: No Built-In Result

JS/TS traditionally uses exceptions or ad-hoc patterns:

```typescript
// TypeScript — no standard Result
// Option A: throw (unchecked, easy to forget)
function divide(a: number, b: number): number {
    if (b === 0) throw new Error("division by zero");
    return a / b;
}

// Option B: neverthrow library
import { ok, err, Result } from "neverthrow";

function divide(a: number, b: number): Result<number, string> {
    if (b === 0) return err("division by zero");
    return ok(a / b);
}
```

Libraries like `neverthrow`, `ts-results`, and `fp-ts` bring Result-like types to TypeScript, but they're opt-in, not enforced by the compiler.

### Comparison Table

| Feature                     | Rust         | Haskell      | Go         | Swift        | TypeScript  |
| --------------------------- | ------------ | ------------ | ---------- | ------------ | ----------- |
| Type                        | `Result<T,E>`| `Either a b` | `(T, err)` | `Result`     | None (libs) |
| Compiler-enforced           | ✅ Yes       | ✅ Yes       | ❌ No      | ✅ Yes       | ❌ No       |
| Can ignore error            | ❌ No¹       | ❌ No        | ✅ Yes     | ❌ No        | ✅ Yes      |
| Zero-cost at runtime        | ✅ Yes       | ❌ Boxed     | ✅ Yes²    | ✅ Yes       | ❌ No       |
| Rich combinators            | ✅ Yes       | ✅ Yes       | ❌ No      | ⚠️ Limited   | ⚠️ Via libs |
| Pattern matching            | ✅ Yes       | ✅ Yes       | ❌ No      | ✅ Yes       | ❌ No       |
| Propagation operator        | `?`          | `do` / `>>=` | `if err`   | `try`/`throws`| `await`³   |

¹ Rust warns on unused `Result` via `#[must_use]` — you can explicitly `let _ = ...` to discard.  
² Go's `error` is an interface — involves dynamic dispatch.  
³ `await` handles Promise rejection, not general Result propagation.

---

## Deep Dive: How Result Is Zero-Cost

One of Rust's core promises is **zero-cost abstractions** — you don't pay for what you don't use, and what you do use is as efficient as hand-written code. `Result` lives up to this completely.

### Memory Layout

`Result<T, E>` is an enum, and Rust enums are stored as a **discriminant tag** + the **data of the active variant**:

```
┌──────────┬──────────────────────────────────────┐
│ tag (u8) │ payload: max(size_of::<T>(), size_of::<E>()) │
└──────────┴──────────────────────────────────────┘
```

```rust
use std::mem::size_of;

fn main() {
    // Result<u32, u32>: tag (+ padding) + max(4, 4) = 8 bytes
    println!("{}", size_of::<Result<u32, u32>>());  // 8

    // Result<u8, u8>: tag + max(1, 1) = 2 bytes
    println!("{}", size_of::<Result<u8, u8>>());    // 2

    // Result<u64, u8>: tag (+ padding) + max(8, 1) = 16 bytes
    println!("{}", size_of::<Result<u64, u8>>());   // 16
}
```

No heap allocation. No vtable. No pointer indirection. The entire `Result` lives on the stack (or inline in its parent struct).

### Niche Optimization

The Rust compiler is clever about finding "impossible" bit patterns and using them as the tag. This is called **niche optimization** (or "niche filling").

The classic example: `Result<T, ()>` where `T` has a niche.

```rust
use std::mem::size_of;

fn main() {
    // A reference can never be null, so null = Err(())
    assert_eq!(size_of::<&i32>(), 8);               // 8 bytes (on 64-bit)
    assert_eq!(size_of::<Result<&i32, ()>>(), 8);    // ALSO 8 bytes!

    // NonZero types have a niche at 0
    use std::num::NonZeroU32;
    assert_eq!(size_of::<NonZeroU32>(), 4);
    assert_eq!(size_of::<Result<NonZeroU32, ()>>(), 4); // same size!

    // Compare with Option — same optimization:
    assert_eq!(size_of::<Option<&i32>>(), 8);        // 8, not 16
}
```

The compiler sees that `&i32` can never be the null pointer (`0x0`), so it uses `0x0` to represent `Err(())`. No extra tag byte needed — the `Result` is **the same size** as the inner type.

### No Dynamic Dispatch

Unlike exception systems in Java/C++/Python, `Result`:

- Does **not** unwind the stack
- Does **not** allocate on the heap
- Does **not** use dynamic dispatch or vtables
- Is **not** a special control flow mechanism

It's just a return value. The function puts `Ok(value)` or `Err(error)` on the stack and returns. The caller pattern-matches on it. That's it.

```rust
// This compiles to roughly the same assembly as a C function
// that returns a struct { bool success; int value; }
fn parse_int(s: &str) -> Result<i32, ()> {
    match s {
        "42" => Ok(42),
        _ => Err(()),
    }
}
```

### Comparison to Exceptions

| Aspect                | Rust `Result`       | C++/Java Exceptions     |
| --------------------- | ------------------- | ----------------------- |
| Memory allocation     | None (stack)        | Heap (exception object) |
| Happy path cost       | Zero                | Near zero¹              |
| Error path cost       | Zero (it's a return)| Very high (stack unwind)|
| Code bloat            | Minimal             | Unwind tables           |
| Predictable timing    | ✅ Yes              | ❌ No                   |

¹ Exceptions have near-zero cost on the "happy path" in table-based implementations, but the unwind tables still increase binary size and the error path is extremely expensive.

`Result` makes both the happy path **and** the error path cheap and predictable. This is why Rust is popular in systems programming, embedded systems, and game engines — you can reason about exactly what the CPU is doing.

---

## Exercises

### Exercise 1: Config Parser Pipeline

Build a function that parses a `"key=value"` config line using only combinators (no `match`, no `if let`, no `?`):

```rust
#[derive(Debug, PartialEq)]
struct ConfigEntry {
    key: String,
    value: i32,
}

/// Parse a line like "timeout=30" into a ConfigEntry.
/// 
/// Errors:
/// - "missing '='" if there's no '=' in the line
/// - "empty key" if the key part is empty
/// - "invalid value: <parse_error>" if the value isn't a valid i32
fn parse_config_line(line: &str) -> Result<ConfigEntry, String> {
    // TODO: Implement using only combinators: map, map_err, and_then, ok_or, etc.
    // Hint: str::split_once('=') returns Option<(&str, &str)>
    todo!()
}

fn main() {
    assert_eq!(
        parse_config_line("timeout=30"),
        Ok(ConfigEntry { key: "timeout".into(), value: 30 })
    );
    assert_eq!(
        parse_config_line("no_equals_sign"),
        Err("missing '='".into())
    );
    assert_eq!(
        parse_config_line("=42"),
        Err("empty key".into())
    );
    assert_eq!(
        parse_config_line("port=abc"),
        Err("invalid value: invalid digit found in string".into())
    );

    println!("All assertions passed!");
}
```

<details>
<summary>💡 Hint</summary>

Start with `line.split_once('=')`, convert the `Option` to `Result` with `ok_or`, then chain with `and_then` for further validation.

</details>

<details>
<summary>✅ Solution</summary>

```rust
fn parse_config_line(line: &str) -> Result<ConfigEntry, String> {
    line.split_once('=')
        .ok_or_else(|| "missing '='".to_string())
        .and_then(|(key, val)| {
            if key.is_empty() {
                Err("empty key".to_string())
            } else {
                Ok((key.to_string(), val))
            }
        })
        .and_then(|(key, val)| {
            val.trim()
                .parse::<i32>()
                .map(|value| ConfigEntry { key, value })
                .map_err(|e| format!("invalid value: {e}"))
        })
}
```

</details>

---

### Exercise 2: Collect and Report

Given a list of strings, parse them all as `f64` values. If **all** parse successfully, return their sum. If **any** fail, return a `Vec` of error messages describing each failure (with the index and original string).

```rust
/// Parse all strings to f64 and return their sum.
/// On failure, return every individual error with its index.
///
/// Example errors: ["index 1: 'abc' is not a valid f64", "index 3: '' is not a valid f64"]
fn sum_or_errors(inputs: &[&str]) -> Result<f64, Vec<String>> {
    // TODO: Implement this.
    // Hint: iterate, parse each, track index. Consider partition or a manual loop.
    todo!()
}

fn main() {
    assert_eq!(sum_or_errors(&["1.5", "2.5", "3.0"]), Ok(7.0));

    let err = sum_or_errors(&["1.0", "abc", "3.0", ""]).unwrap_err();
    assert_eq!(err.len(), 2);
    assert!(err[0].contains("index 1"));
    assert!(err[1].contains("index 3"));

    println!("All assertions passed!");
}
```

<details>
<summary>💡 Hint</summary>

Iterate with `.enumerate()`, collect parse results, then partition into successes and failures. If failures is empty, sum the successes. Otherwise, return the failure messages.

</details>

<details>
<summary>✅ Solution</summary>

```rust
fn sum_or_errors(inputs: &[&str]) -> Result<f64, Vec<String>> {
    let mut values = Vec::new();
    let mut errors = Vec::new();

    for (i, s) in inputs.iter().enumerate() {
        match s.parse::<f64>() {
            Ok(v) => values.push(v),
            Err(_) => errors.push(format!("index {i}: '{s}' is not a valid f64")),
        }
    }

    if errors.is_empty() {
        Ok(values.iter().sum())
    } else {
        Err(errors)
    }
}
```

</details>

---

### Exercise 3: Option ↔ Result Round Trip

Implement a small user-lookup system that demonstrates converting between `Option` and `Result`:

```rust
use std::collections::HashMap;

struct UserDb {
    users: HashMap<u32, String>,
}

impl UserDb {
    fn new() -> Self {
        let mut users = HashMap::new();
        users.insert(1, "Alice".to_string());
        users.insert(2, "Bob".to_string());
        users.insert(3, "Charlie".to_string());
        UserDb { users }
    }

    /// Look up a user by ID. Returns Option (they might not exist).
    fn find(&self, id: u32) -> Option<&String> {
        self.users.get(&id)
    }

    /// Look up a user AND return their name length.
    /// Convert the Option to a Result with a descriptive error.
    /// Then use map() to compute the length.
    fn name_length(&self, id: u32) -> Result<usize, String> {
        // TODO: use find(), ok_or/ok_or_else, and map
        todo!()
    }

    /// Look up two users and concatenate their names.
    /// Both must exist, or return an error for the first missing one.
    fn concat_names(&self, id1: u32, id2: u32) -> Result<String, String> {
        // TODO: use find(), ok_or_else, and_then/map
        todo!()
    }
}

fn main() {
    let db = UserDb::new();

    assert_eq!(db.name_length(1), Ok(5));   // "Alice" = 5
    assert_eq!(db.name_length(99), Err("user 99 not found".into()));

    assert_eq!(db.concat_names(1, 2), Ok("AliceBob".into()));
    assert_eq!(db.concat_names(1, 99), Err("user 99 not found".into()));

    // Bonus: convert Result back to Option
    let opt: Option<usize> = db.name_length(1).ok();
    assert_eq!(opt, Some(5));

    println!("All assertions passed!");
}
```

<details>
<summary>✅ Solution</summary>

```rust
fn name_length(&self, id: u32) -> Result<usize, String> {
    self.find(id)
        .ok_or_else(|| format!("user {id} not found"))
        .map(|name| name.len())
}

fn concat_names(&self, id1: u32, id2: u32) -> Result<String, String> {
    let name1 = self.find(id1)
        .ok_or_else(|| format!("user {id1} not found"))?;
    let name2 = self.find(id2)
        .ok_or_else(|| format!("user {id2} not found"))?;
    Ok(format!("{name1}{name2}"))
}
```

(Yes, the `concat_names` solution uses `?` — that's fine here. A pure combinator version would use nested `and_then`, which is much harder to read.)

</details>

---

## Summary

| Concept                     | Key Takeaway                                                        |
| --------------------------- | ------------------------------------------------------------------- |
| **Result\<T, E\>**          | Just an enum: `Ok(T)` or `Err(E)`. The compiler enforces handling.  |
| **Pattern matching**        | `match`, `if let`, `let else` — the foundation of Result handling.  |
| **Combinators**             | `map`, `map_err`, `and_then`, `or_else` — pipeline-style transforms.|
| **unwrap / expect**         | Panic on Err. Use `expect` with context; avoid `unwrap` in prod.    |
| **Std library usage**       | File I/O, parsing, networking — Result is everywhere.               |
| **Option ↔ Result**         | `ok_or` / `ok_or_else` converts Option → Result. `.ok()` goes back. |
| **Collecting Results**      | `Iterator<Result<T,E>>` → `Result<Vec<T>, E>` with short-circuit.  |
| **Zero-cost**               | Stack-allocated, no dynamic dispatch, niche-optimized.              |
| **vs other languages**      | Compiler-enforced + zero-cost — a combination unique to Rust.       |

`Result` is the backbone of error handling in Rust. Master the combinators and you'll find yourself writing clean, expressive code where errors are never ignored and always handled intentionally.

**Next up:** The `?` operator — Rust's syntactic sugar that makes `Result` chains as clean as exception-based code, without any of the downsides.

---

**Previous:** [← panic!](./02-panic.md) · **Next:** [The ? Operator →](./04-question-mark-operator.md)

<p align="center"><i>Tutorial 3 of 7 — Stage 7: Error Handling</i></p>

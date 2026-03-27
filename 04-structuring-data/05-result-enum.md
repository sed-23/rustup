# The Result Enum ⚖️

> **`Result<T, E>` is how Rust handles operations that can fail. Instead of exceptions, Rust returns a value that's either Ok(success) or Err(failure) — and the compiler makes you handle both.**

---

## Table of Contents

- [Exceptions vs Results](#exceptions-vs-results)
- [What Is Result\<T, E\>?](#what-is-resultt-e)
- [Creating Results](#creating-results)
- [Handling Results](#handling-results)
  - [match](#match)
  - [unwrap and expect](#unwrap-and-expect)
  - [unwrap_or](#unwrap_or)
  - [The ? Operator](#the--operator)
- [Result Methods](#result-methods)
- [Converting Between Result and Option](#converting-between-result-and-option)
- [Functions That Return Result](#functions-that-return-result)
- [Common Result Patterns](#common-result-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Exceptions vs Results

```
Java/Python:
  try {
      let file = open("config.txt");   // Might throw IOException
      let data = file.read();           // Might throw IOException
  } catch (IOException e) {
      // Handle error
  }
  // Problem: nothing in the function signature tells you it can fail!

Rust:
  fn open(path: &str) -> Result<File, io::Error>
  // ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  // The return type TELLS you it can fail, and how!
```

Benefits of Result:
- **Explicit** — the type signature shows that failure is possible
- **Forced handling** — the compiler warns if you ignore a Result
- **No hidden control flow** — no exceptions unwinding the stack

---

### The History of Error Handling in Programming

Rust's `Result<T, E>` didn't appear out of thin air. It's the culmination of **60+ years** of trial and error (pun intended) in how programming languages deal with things going wrong.

#### The Timeline

```
1950s-60s ─── Error Codes (integers)
     │        UNIX convention: 0 = success, nonzero = error
     │        Simple, but nothing forces you to check them.
     │
1970s ─────── C: errno global variable
     │        System calls set errno; you check it after the call.
     │        Problem: easy to forget. Silent bugs everywhere.
     │
1990 ──────── C++: throw / try / catch
     │        Powerful but expensive — stack unwinding is unpredictable.
     │        "Invisible" control flow: any function might throw.
     │
1995 ──────── Java: Checked exceptions
     │        Compiler forces you to handle or declare exceptions.
     │        Good idea, but led to the "catch and ignore" anti-pattern:
     │          catch (Exception e) { /* TODO */ }
     │
1995-2010 ─── Python / Ruby / JavaScript: Unchecked exceptions
     │        Runtime errors propagate up the call stack.
     │        No compile-time safety whatsoever.
     │
2009 ──────── Go: Multiple return values (value, err)
     │        Explicit, but verbose:  if err != nil  on every other line.
     │
~2000s ────── Haskell: Either a b
     │        The functional approach — a sum type, just like Result.
     │        Type-safe, composable, but needs monadic syntax.
     │
2015 ──────── Rust: Result<T, E> + the ? operator (added 1.13, 2016)
              Combines Go's explicitness with Haskell's type safety,
              plus ergonomic sugar that makes it practical at scale.
```

#### The Key Insight

Rust makes errors **part of the type system**. A function that returns `Result<File, io::Error>` can't be called without acknowledging the possibility of failure. The compiler literally won't let you:

```rust
// This produces a compiler WARNING:
fn main() {
    std::fs::read_to_string("file.txt");  // unused `Result` that must be used
}
```

In C, Java, or Python, forgetting to handle an error compiles silently and waits to become a production incident.

#### Comparison Table

| Language | Error Mechanism | Compile-Time Checked? | Performance Cost | Can Be Ignored? |
|---|---|---|---|---|
| C | `errno` / return codes | No | None | Yes — silently |
| C++ | `throw` / `try` / `catch` | No | Stack unwinding | Yes — uncaught |
| Java | Checked exceptions | Partially | Stack unwinding | Yes — catch & ignore |
| Python/JS | Unchecked exceptions | No | Stack unwinding | Yes — crashes at runtime |
| Go | `(value, err)` tuple | No | None | Yes — `_ , _ = f()` |
| Haskell | `Either a b` | Yes | None (lazy) | No (type-enforced) |
| **Rust** | **`Result<T, E>`** | **Yes** | **None (zero-cost)** | **No — compiler warning** |

> Every generation of languages learned from the last. Rust's `Result` is the current best answer — explicit, zero-cost, and impossible to accidentally ignore.

---

## What Is `Result<T, E>`?

```rust
// Defined in the standard library:
enum Result<T, E> {
    Ok(T),    // Success, with value of type T
    Err(E),   // Failure, with error of type E
}
```

- `T` = the success type
- `E` = the error type
- `Result<i32, String>` = "either an i32 or a String error message"

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    println!("{:?}", divide(10.0, 2.0));   // Ok(5.0)
    println!("{:?}", divide(10.0, 0.0));   // Err("Division by zero")
}
```

---

## Creating Results

```rust
fn main() {
    // Explicit
    let success: Result<i32, String> = Ok(42);
    let failure: Result<i32, String> = Err(String::from("something went wrong"));
    
    // From parsing
    let parsed: Result<i32, _> = "42".parse();     // Ok(42)
    let failed: Result<i32, _> = "abc".parse();    // Err(ParseIntError)
    
    // From file operations
    use std::fs;
    let content: Result<String, _> = fs::read_to_string("hello.txt");
}
```

---

## Handling Results

### `match`

```rust
fn main() {
    let result = divide(10.0, 3.0);
    
    match result {
        Ok(value) => println!("Result: {:.2}", value),
        Err(e) => println!("Error: {}", e),
    }
}
```

### `unwrap` and `expect`

```rust
fn main() {
    let x: Result<i32, String> = Ok(42);
    println!("{}", x.unwrap());  // 42
    
    let y: Result<i32, String> = Err(String::from("oops"));
    // println!("{}", y.unwrap());  // PANIC: called `Result::unwrap()` on an `Err` value: "oops"
    
    // expect: custom panic message
    let z: Result<i32, String> = Ok(5);
    println!("{}", z.expect("failed to get value"));  // 5
}
```

### `unwrap_or`

```rust
fn main() {
    let x: Result<i32, String> = Err(String::from("error"));
    
    let val = x.unwrap_or(0);                    // 0
    let val = x.unwrap_or_else(|e| {
        println!("Error occurred: {}", e);
        -1
    });
    let val = x.unwrap_or_default();             // 0 (i32::default())
}
```

### The `?` Operator

The `?` propagates errors automatically. If the Result is `Err`, the function returns that error immediately:

```rust
use std::fs;
use std::io;

fn read_username() -> Result<String, io::Error> {
    let content = fs::read_to_string("username.txt")?;
    //                                               ^ if Err, return it
    Ok(content.trim().to_string())
}

// Without ?, you'd write:
fn read_username_verbose() -> Result<String, io::Error> {
    let content = match fs::read_to_string("username.txt") {
        Ok(c) => c,
        Err(e) => return Err(e),
    };
    Ok(content.trim().to_string())
}
```

Chaining with `?`:

```rust
use std::fs;
use std::io;

fn first_line_of_file(path: &str) -> Result<String, io::Error> {
    let content = fs::read_to_string(path)?;
    let first = content.lines().next().unwrap_or("").to_string();
    Ok(first)
}
```

---

### The `?` Operator Deep Dive — Syntactic Sugar with Superpowers

The `?` operator looks small, but it's doing a lot of heavy lifting behind the scenes. Let's crack it open.

#### What `?` Actually Expands To

When you write:

```rust
let x = some_result?;
```

The compiler transforms it into roughly:

```rust
let x = match some_result {
    Ok(val) => val,
    Err(e) => return Err(e.into()),   // Note the .into()!
};
```

That `.into()` call is crucial — it means `?` **automatically converts error types** using the `From` trait. If your function returns `Result<T, MyError>` and the expression produces a `std::io::Error`, Rust will call `MyError::from(io_error)` for you.

```
  expression?
      │
      ▼
  ┌─────────────┐
  │ Is it Ok(v)? │──── Yes ──→ unwrap v, continue
  └──────┬──────┘
         │ No (it's Err(e))
         ▼
  ┌──────────────────┐
  │ e.into() converts │
  │ to return type's E │
  └──────┬───────────┘
         ▼
    return Err(converted_e)
```

#### Before `?` — The `try!()` Macro

Before Rust 1.13 (November 2016), you had to use the `try!()` macro:

```rust
// Old style (deprecated):
fn read_username() -> Result<String, io::Error> {
    let content = try!(fs::read_to_string("username.txt"));
    Ok(content.trim().to_string())
}

// New style (Rust 1.13+):
fn read_username() -> Result<String, io::Error> {
    let content = fs::read_to_string("username.txt")?;
    Ok(content.trim().to_string())
}
```

The `?` operator was a watershed moment for Rust ergonomics — it made `Result`-based error handling practical for everyday code.

#### `?` Works on Option Too

```rust
fn first_even(numbers: &[i32]) -> Option<i32> {
    let first = numbers.first()?;   // Returns None if slice is empty
    if first % 2 == 0 {
        Some(*first)
    } else {
        None
    }
}
```

On `Option`, `?` returns `None` early instead of `Err(e)`.

#### Chaining `?` for Concise Pipelines

You can chain multiple `?` operators in a single expression:

```rust
// Parse a number from a file — two fallible operations in one line:
let count = std::fs::read_to_string("count.txt")?.trim().parse::<i32>()?;
```

Without `?`, this would be ~10 lines of nested match expressions.

#### How Other Languages Handle Early Returns

| Language | Early-Return Pattern | Lines per error check |
|---|---|---|
| **Rust** | `let x = expr?;` | **1** |
| Go | `if err != nil { return err }` | 3 |
| Swift | `guard let x = try? expr else { return }` | 1-2 |
| Kotlin | `val x = expr ?: return` | 1 |
| Java | `try { ... } catch (...) { throw ... }` | 3-5 |

The `?` operator is what makes `Result`-based error handling **practical** at scale. Without it, Rust would have Go-level verbosity — explicit but tedious. With it, you get type-safe error propagation in a single character.

---

## Result Methods

```rust
fn main() {
    let ok: Result<i32, String> = Ok(5);
    let err: Result<i32, String> = Err("error".into());
    
    // Check which variant
    println!("{}", ok.is_ok());    // true
    println!("{}", ok.is_err());   // false
    
    // Map the success value
    let doubled = ok.map(|n| n * 2);     // Ok(10)
    let doubled = err.map(|n| n * 2);    // Still Err("error")
    
    // Map the error
    let mapped = err.map_err(|e| format!("Error: {}", e));  // Err("Error: error")
    
    // Convert to Option
    let opt: Option<i32> = ok.ok();     // Some(5)
    let opt: Option<i32> = err.ok();    // None
    let opt: Option<String> = err.err(); // Some("error")
    
    // and_then: chain operations that return Result
    let result = ok.and_then(|n| {
        if n > 0 { Ok(n * 10) } else { Err("negative".into()) }
    });
}
```

---

## Converting Between Result and Option

```rust
fn main() {
    // Result → Option (discard error info)
    let r: Result<i32, String> = Ok(5);
    let o: Option<i32> = r.ok();       // Some(5)
    
    let r: Result<i32, String> = Err("bad".into());
    let o: Option<i32> = r.ok();       // None
    
    // Option → Result (add error info)
    let o: Option<i32> = Some(5);
    let r: Result<i32, String> = o.ok_or(String::from("was None"));  // Ok(5)
    
    let o: Option<i32> = None;
    let r: Result<i32, String> = o.ok_or(String::from("was None"));  // Err("was None")
    
    // ok_or_else: lazy error
    let r = o.ok_or_else(|| String::from("computed error"));
}
```

---

## Functions That Return Result

```rust
use std::num::ParseIntError;

fn parse_and_double(s: &str) -> Result<i32, ParseIntError> {
    let n = s.parse::<i32>()?;
    Ok(n * 2)
}

fn parse_two_numbers(a: &str, b: &str) -> Result<(i32, i32), ParseIntError> {
    let x = a.parse::<i32>()?;
    let y = b.parse::<i32>()?;
    Ok((x, y))
}

fn main() {
    println!("{:?}", parse_and_double("21"));    // Ok(42)
    println!("{:?}", parse_and_double("abc"));   // Err(invalid digit)
    println!("{:?}", parse_two_numbers("1", "2")); // Ok((1, 2))
}
```

### Using Result in main()

```rust
use std::num::ParseIntError;

fn main() -> Result<(), ParseIntError> {
    let n: i32 = "42".parse()?;
    println!("Parsed: {}", n);
    Ok(())
}
```

---

## Common Result Patterns

### Early Return on Error

```rust
fn process_data(input: &str) -> Result<String, String> {
    if input.is_empty() {
        return Err(String::from("Input cannot be empty"));
    }
    
    let trimmed = input.trim();
    if trimmed.len() < 3 {
        return Err(String::from("Input too short"));
    }
    
    Ok(format!("Processed: {}", trimmed.to_uppercase()))
}
```

### Collecting Results

```rust
fn main() {
    let strings = vec!["1", "2", "three", "4"];
    
    // Collect into Result<Vec<i32>, _> — stops at first error
    let result: Result<Vec<i32>, _> = strings.iter()
        .map(|s| s.parse::<i32>())
        .collect();
    
    println!("{:?}", result);  // Err(invalid digit found in string)
    
    // Partition into successes and failures
    let (oks, errs): (Vec<_>, Vec<_>) = strings.iter()
        .map(|s| s.parse::<i32>())
        .partition(Result::is_ok);
    
    let values: Vec<i32> = oks.into_iter().map(Result::unwrap).collect();
    println!("Parsed: {:?}", values);  // [1, 2, 4]
}
```

---

### Custom Error Types and the Error Ecosystem

In the examples above, we used `String` as our error type. That works for learning, but **real Rust projects almost never use `String` for errors**. Here's what the real world looks like.

#### The Standard Pattern: Error Enums

Define an `enum` with one variant per error case:

```rust
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
    NotFound(String),
}

// Implement Display (required for the Error trait)
impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO error: {}", e),
            AppError::Parse(e) => write!(f, "Parse error: {}", e),
            AppError::NotFound(name) => write!(f, "Not found: {}", name),
        }
    }
}

// Implement From so the ? operator can auto-convert
impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self { AppError::Io(e) }
}
impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self { AppError::Parse(e) }
}
```

This is correct but verbose. That's where the ecosystem helps.

#### The `std::error::Error` Trait

```
   std::error::Error
   ├── requires: Display + Debug
   ├── fn source(&self) -> Option<&dyn Error>
   │   └── returns the underlying cause (for error chains)
   └── automatically implemented by many std types:
       ├── io::Error
       ├── ParseIntError
       ├── ParseFloatError
       └── ... many more
```

The `source()` method enables **error chains** — when one error causes another, you can walk the chain to find the root cause.

#### The Crate Ecosystem

Two crates dominate Rust error handling:

| Crate | Purpose | Use When |
|---|---|---|
| **`thiserror`** | Derive macro for custom error enums | Writing a **library** — callers need specific error variants |
| **`anyhow`** | Catch-all error type + context | Writing an **application** — you just want to propagate & report |

With `thiserror`, the boilerplate above becomes:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("Not found: {0}")]
    NotFound(String),
}
// That's it — Display, Debug, Error, and From are all derived.
```

With `anyhow`, you skip custom types entirely:

```rust
use anyhow::{Context, Result};

fn read_config() -> Result<i32> {   // anyhow::Result<T> = Result<T, anyhow::Error>
    let text = std::fs::read_to_string("config.txt")
        .context("while reading config file")?;    // adds context to error chain
    let value = text.trim().parse::<i32>()
        .context("while parsing config value")?;
    Ok(value)
}
```

#### When to Use Which

```
  Are you writing a library that others will depend on?
      │
      ├── Yes ──→ Use thiserror (callers need to match on specific errors)
      │
      └── No (it's an application) ──→ Use anyhow (just propagate & display)
```

> **Real-world note:** The Rust error handling ecosystem is still evolving. Don't worry about getting it perfect on day one — start with `String` errors while learning, graduate to `thiserror`/`anyhow` when you're comfortable with `Result`. The important thing is that `Result` exists and the compiler has your back.

---

## Exercises

### Exercise 1: Parse Config

Write a function that parses "key=value" strings:

```rust
fn parse_kv(input: &str) -> Result<(&str, &str), String> {
    // Your code
}

fn main() {
    assert_eq!(parse_kv("name=Alice"), Ok(("name", "Alice")));
    assert!(parse_kv("invalid").is_err());
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn parse_kv(input: &str) -> Result<(&str, &str), String> {
    match input.split_once('=') {
        Some((key, value)) => Ok((key, value)),
        None => Err(format!("No '=' found in '{}'", input)),
    }
}
```

</details>

### Exercise 2: Chain Results

Write a function that reads a number from a string, doubles it, and converts back to string — all with `?`:

```rust
fn double_string_number(s: &str) -> Result<String, std::num::ParseIntError> {
    // Your code
}
```

<details>
<summary>Solution</summary>

```rust
fn double_string_number(s: &str) -> Result<String, std::num::ParseIntError> {
    let n: i32 = s.parse()?;
    Ok((n * 2).to_string())
}
```

</details>

---

## Summary

| Concept | Option | Result |
|---------|--------|--------|
| Variants | `Some(T)` / `None` | `Ok(T)` / `Err(E)` |
| Meaning | Present or absent | Success or failure |
| Error info | No | Yes (the E type) |
| `?` operator | Returns `None` | Returns `Err(e)` |
| Use when | Value might not exist | Operation might fail |

**Key Takeaways:**
1. `Result<T, E>` = `Ok(T)` for success, `Err(E)` for failure
2. The `?` operator propagates errors — cleaner than manual match
3. The compiler warns you if you ignore a `Result`
4. Use `map`, `and_then`, `map_err` to transform Results without unwrapping
5. This is a preview — Stage 7 covers error handling in depth

---

**Next:** [Pattern Matching with match →](./06-pattern-matching.md)

<p align="center"><i>Tutorial 5 of 8 — Stage 4: Structuring Data</i></p>

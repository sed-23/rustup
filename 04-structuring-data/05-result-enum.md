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

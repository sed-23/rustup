# The Option Enum 🎁

> **Rust has no null. Instead, it uses `Option<T>` — a type that explicitly represents "a value or nothing." This eliminates null pointer exceptions entirely.**

---

## Table of Contents

- [The Billion-Dollar Mistake](#the-billion-dollar-mistake)
- [What Is Option\<T\>?](#what-is-optiont)
- [Creating Option Values](#creating-option-values)
- [Why Option Is Better Than Null](#why-option-is-better-than-null)
- [Getting the Value Out](#getting-the-value-out)
  - [match](#match)
  - [if let](#if-let)
  - [unwrap and expect](#unwrap-and-expect)
  - [unwrap_or and unwrap_or_else](#unwrap_or-and-unwrap_or_else)
  - [The ? Operator (Preview)](#the--operator-preview)
- [Option Methods](#option-methods)
  - [is_some and is_none](#is_some-and-is_none)
  - [map](#map)
  - [and_then (Flat Map)](#and_then-flat-map)
  - [or and or_else](#or-and-or_else)
  - [filter](#filter)
- [Option in Practice](#option-in-practice)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Billion-Dollar Mistake

Tony Hoare, inventor of null references, called them his "billion-dollar mistake":

```
C/C++:     NULL pointer → segfault, undefined behavior
Java:      null → NullPointerException at runtime
Python:    None → AttributeError at runtime
JavaScript: null/undefined → TypeError at runtime

Rust:      Option<T> → compiler forces you to handle it ✅
```

Rust doesn't have null at all. If a value might be absent, the type system **requires** you to deal with that possibility.

---

## What Is `Option<T>`?

`Option` is a standard library enum with exactly two variants:

```rust
// This is defined in the standard library:
enum Option<T> {
    Some(T),   // There IS a value of type T
    None,      // There is NO value
}
```

`T` is a **generic type parameter** — it works with any type. `Option<i32>` means "maybe an integer." `Option<String>` means "maybe a string."

```rust
fn main() {
    let some_number: Option<i32> = Some(42);
    let no_number: Option<i32> = None;
    
    let some_text: Option<String> = Some(String::from("hello"));
    let no_text: Option<String> = None;
}
```

`Option`, `Some`, and `None` are in the **prelude** — no imports needed.

---

## Creating Option Values

```rust
fn main() {
    // Explicit type annotation
    let x: Option<i32> = Some(5);
    let y: Option<i32> = None;
    
    // Type inferred from context
    let a = Some(42);          // Option<i32>
    let b = Some("hello");    // Option<&str>
    let c = Some(3.14);       // Option<f64>
    
    // None needs type annotation (Rust can't infer T)
    let d: Option<i32> = None;
    // let e = None;  // ❌ Error: type annotations needed
    
    // From methods that might fail
    let v = vec![1, 2, 3];
    let first: Option<&i32> = v.first();     // Some(&1)
    let tenth: Option<&i32> = v.get(10);     // None
    
    let text = "42";
    let parsed: Option<i32> = text.parse().ok();  // Some(42)
    
    let text = "abc";
    let parsed: Option<i32> = text.parse().ok();  // None
}
```

---

## Why Option Is Better Than Null

The key difference: **you can't accidentally use an Option as if it has a value.**

```rust
fn main() {
    let x: i32 = 5;
    let y: Option<i32> = Some(5);
    
    // let sum = x + y;  // ❌ Can't add i32 and Option<i32>!
    
    // You MUST extract the value first:
    let sum = x + y.unwrap();  // ✅ But might panic if None
    
    // Better: handle the None case
    let sum = match y {
        Some(val) => x + val,
        None => x,  // Default to just x
    };
    
    println!("{}", sum);  // 10
}
```

**The compiler forces you to check.** You physically cannot use `Option<T>` where `T` is expected without unwrapping it first.

---

## Getting the Value Out

### `match`

The most thorough way — handles both cases:

```rust
fn describe_number(n: Option<i32>) -> String {
    match n {
        Some(val) => format!("The number is {}", val),
        None => String::from("No number"),
    }
}

fn main() {
    println!("{}", describe_number(Some(42)));   // "The number is 42"
    println!("{}", describe_number(None));        // "No number"
}
```

### `if let`

When you only care about the `Some` case:

```rust
fn main() {
    let name: Option<String> = Some(String::from("Alice"));
    
    if let Some(n) = &name {
        println!("Hello, {}!", n);
    }
    // No else needed — we just skip for None
    
    // With else:
    if let Some(n) = &name {
        println!("Name: {}", n);
    } else {
        println!("Anonymous user");
    }
}
```

### `unwrap` and `expect`

**Quick and dirty — panic if None:**

```rust
fn main() {
    let x: Option<i32> = Some(42);
    let val = x.unwrap();       // 42
    
    let y: Option<i32> = None;
    // let val = y.unwrap();    // PANIC: called `Option::unwrap()` on a `None` value
    
    // expect: same but with a custom message
    let z: Option<i32> = Some(5);
    let val = z.expect("z should have a value");  // 5
    
    // let val = y.expect("y should have a value");
    // PANIC: y should have a value
}
```

**Use `unwrap`/`expect` when:**
- You're prototyping and don't want to handle errors yet
- You've already checked that the value is `Some`
- A `None` would be a programming bug (not a runtime condition)

### `unwrap_or` and `unwrap_or_else`

Provide a default value:

```rust
fn main() {
    let x: Option<i32> = None;
    
    // unwrap_or: provide a default value
    let val = x.unwrap_or(0);
    println!("{}", val);  // 0
    
    // unwrap_or_else: compute default lazily
    let val = x.unwrap_or_else(|| {
        println!("Computing default...");
        42
    });
    println!("{}", val);  // 42
    
    // unwrap_or_default: use the type's Default implementation
    let val: i32 = x.unwrap_or_default();  // 0 (i32 default)
    let val: String = None::<String>.unwrap_or_default();  // "" (String default)
}
```

### The `?` Operator (Preview)

In functions that return `Option`, you can use `?` to propagate `None`:

```rust
fn both_positive(a: Option<i32>, b: Option<i32>) -> Option<i32> {
    let x = a?;  // Return None if a is None
    let y = b?;  // Return None if b is None
    
    if x > 0 && y > 0 {
        Some(x + y)
    } else {
        None
    }
}

fn main() {
    println!("{:?}", both_positive(Some(3), Some(5)));   // Some(8)
    println!("{:?}", both_positive(Some(3), None));       // None
    println!("{:?}", both_positive(Some(-1), Some(5)));   // None
}
```

---

## Option Methods

### `is_some` and `is_none`

```rust
let x: Option<i32> = Some(5);
let y: Option<i32> = None;

println!("{}", x.is_some());  // true
println!("{}", x.is_none());  // false
println!("{}", y.is_some());  // false
println!("{}", y.is_none());  // true
```

### `map`

Transform the inner value without unwrapping:

```rust
fn main() {
    let x: Option<i32> = Some(5);
    let doubled: Option<i32> = x.map(|n| n * 2);
    println!("{:?}", doubled);  // Some(10)
    
    let y: Option<i32> = None;
    let doubled: Option<i32> = y.map(|n| n * 2);
    println!("{:?}", doubled);  // None (map does nothing on None)
    
    // Chain maps:
    let result = Some("42")
        .map(|s| s.parse::<i32>())   // Some(Ok(42))
        .map(|r| r.ok())             // Some(Some(42))
        ; // Nested! Use and_then instead
}
```

### `and_then` (Flat Map)

Like `map`, but the function itself returns `Option`. Prevents nesting:

```rust
fn parse_positive(s: &str) -> Option<u32> {
    let n: i32 = s.parse().ok()?;
    if n > 0 { Some(n as u32) } else { None }
}

fn main() {
    let input: Option<&str> = Some("42");
    
    // map would give Option<Option<u32>> — nested!
    // and_then flattens it to Option<u32>
    let result = input.and_then(parse_positive);
    println!("{:?}", result);  // Some(42)
    
    let result = Some("-5").and_then(parse_positive);
    println!("{:?}", result);  // None
    
    let result: Option<u32> = None.and_then(parse_positive);
    println!("{:?}", result);  // None
}
```

### `or` and `or_else`

Provide a fallback Option:

```rust
fn main() {
    let a: Option<i32> = None;
    let b: Option<i32> = Some(5);
    
    println!("{:?}", a.or(b));         // Some(5) — a is None, use b
    println!("{:?}", b.or(Some(10)));  // Some(5) — b has value, ignore fallback
    
    // or_else: lazy fallback
    let result = a.or_else(|| Some(99));
    println!("{:?}", result);  // Some(99)
}
```

### `filter`

Keep the value only if it passes a test:

```rust
fn main() {
    let x = Some(4);
    
    let even = x.filter(|n| n % 2 == 0);
    println!("{:?}", even);  // Some(4)
    
    let odd = x.filter(|n| n % 2 != 0);
    println!("{:?}", odd);   // None (4 is not odd)
}
```

---

## Option in Practice

### Finding Elements

```rust
fn find_user(users: &[&str], name: &str) -> Option<usize> {
    users.iter().position(|&u| u == name)
}

fn main() {
    let users = vec!["Alice", "Bob", "Charlie"];
    
    match find_user(&users, "Bob") {
        Some(idx) => println!("Found Bob at index {}", idx),
        None => println!("Bob not found"),
    }
}
```

### Chaining Operations

```rust
fn get_first_even(numbers: &[i32]) -> Option<i32> {
    numbers.iter()
        .find(|&&n| n % 2 == 0)
        .copied()  // Option<&i32> → Option<i32>
}

fn get_street_name(address: &Option<Address>) -> Option<&str> {
    address.as_ref()
        .map(|addr| addr.street.as_str())
}
```

### Config with Optional Fields

```rust
struct Config {
    host: String,
    port: u16,
    database: Option<String>,
    max_retries: Option<u32>,
}

impl Config {
    fn database_url(&self) -> String {
        let db = self.database.as_deref().unwrap_or("default_db");
        format!("postgres://{}:{}/{}", self.host, self.port, db)
    }
    
    fn retries(&self) -> u32 {
        self.max_retries.unwrap_or(3)
    }
}
```

---

## Common Mistakes

### Mistake 1: Unnecessary unwrap

```rust
// ❌ Panics if item not found
let idx = vec![1, 2, 3].iter().position(|&x| x == 5).unwrap();

// ✅ Handle the None case
if let Some(idx) = vec![1, 2, 3].iter().position(|&x| x == 5) {
    println!("Found at {}", idx);
}
```

### Mistake 2: Nested Options

```rust
// ❌ Confusing nesting
let x: Option<Option<i32>> = Some(Some(5));

// ✅ Use and_then to flatten
let x: Option<i32> = Some("5").and_then(|s| s.parse().ok());
```

### Mistake 3: Checking then unwrapping

```rust
let x: Option<i32> = Some(5);

// ❌ Checking then unwrapping — redundant
if x.is_some() {
    let val = x.unwrap();
    println!("{}", val);
}

// ✅ if let — one step
if let Some(val) = x {
    println!("{}", val);
}
```

---

## Exercises

### Exercise 1: Safe Division

Write a function that returns `None` for division by zero:

```rust
fn safe_divide(a: f64, b: f64) -> Option<f64> {
    // Your code
}

fn main() {
    assert_eq!(safe_divide(10.0, 2.0), Some(5.0));
    assert_eq!(safe_divide(10.0, 0.0), None);
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn safe_divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}
```

</details>

### Exercise 2: Chaining Options

Given a list of strings, find the first one that parses to a positive integer:

```rust
fn first_positive(items: &[&str]) -> Option<u32> {
    // Your code
}

fn main() {
    assert_eq!(first_positive(&["abc", "-5", "0", "42", "7"]), Some(42));
    assert_eq!(first_positive(&["abc", "xyz"]), None);
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn first_positive(items: &[&str]) -> Option<u32> {
    items.iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .find(|&n| n > 0)
        .map(|n| n as u32)
}
```

</details>

---

## Summary

| Method | What It Does | None Behavior |
|--------|-------------|---------------|
| `unwrap()` | Get value or panic | Panic |
| `expect("msg")` | Get value or panic with message | Panic with message |
| `unwrap_or(val)` | Get value or use default | Return `val` |
| `unwrap_or_else(f)` | Get value or compute default | Return `f()` |
| `unwrap_or_default()` | Get value or type's default | Return `T::default()` |
| `map(f)` | Transform inner value | None |
| `and_then(f)` | Transform, flatten nested Option | None |
| `filter(pred)` | Keep Some if predicate passes | None |
| `or(other)` | Use fallback Option | Return `other` |
| `is_some()` / `is_none()` | Check variant | true for `is_none` |

**Key Takeaways:**
1. Rust has no null — use `Option<T>` instead
2. `Some(value)` means present, `None` means absent
3. The compiler **forces** you to handle both cases
4. `match` and `if let` are the primary ways to extract values
5. `map` and `and_then` let you chain operations elegantly
6. Use `unwrap`/`expect` only when you're sure or prototyping

---

**Next:** [The Result Enum →](./05-result-enum.md)

<p align="center"><i>Tutorial 4 of 8 — Stage 4: Structuring Data</i></p>

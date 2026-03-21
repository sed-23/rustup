# Pattern Matching with match 🎯

> **`match` is one of Rust's most powerful features. It compares a value against a series of patterns and runs the code for the first match. The compiler guarantees you handle every possible case.**

---

## Table of Contents

- [What Is match?](#what-is-match)
- [Basic Syntax](#basic-syntax)
- [match Is an Expression](#match-is-an-expression)
- [Exhaustiveness](#exhaustiveness)
- [Pattern Types](#pattern-types)
  - [Literal Patterns](#literal-patterns)
  - [Variable Binding](#variable-binding)
  - [Wildcard _](#wildcard-_)
  - [Multiple Patterns with |](#multiple-patterns-with-)
  - [Range Patterns](#range-patterns)
  - [Enum Patterns](#enum-patterns)
  - [Struct Patterns](#struct-patterns)
  - [Tuple Patterns](#tuple-patterns)
  - [Reference Patterns](#reference-patterns)
- [Match Guards](#match-guards)
- [@ Bindings](#-bindings)
- [Nested Matching](#nested-matching)
- [Common Patterns](#common-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is `match`?

```rust
fn main() {
    let number = 3;
    
    match number {
        1 => println!("One"),
        2 => println!("Two"),
        3 => println!("Three"),
        _ => println!("Something else"),
    }
}
```

Think of `match` as a more powerful version of **switch/case** from other languages — but with pattern matching and guaranteed exhaustiveness.

---

## Basic Syntax

```rust
match value {
    pattern1 => expression1,
    pattern2 => expression2,
    pattern3 => {
        // Multi-line code in a block
        let x = compute();
        x + 1
    },
    _ => default_expression,
}
```

Rules:
- Each `pattern => expression` is called an **arm**
- Arms are checked **top to bottom** — first match wins
- The `_` wildcard matches everything
- Each arm separated by commas (optional for blocks)

---

## `match` Is an Expression

`match` returns a value. All arms must return the same type:

```rust
fn main() {
    let x = 5;
    
    let description = match x {
        1 => "one",
        2 => "two",
        3..=5 => "three to five",
        _ => "something else",
    };
    
    println!("{}", description);  // "three to five"
}
```

---

## Exhaustiveness

The compiler ensures you handle **every possible value**. This is called **exhaustive matching**:

```rust
enum Color {
    Red,
    Green,
    Blue,
}

fn describe(c: Color) -> &'static str {
    match c {
        Color::Red => "red",
        Color::Green => "green",
        // ❌ ERROR: non-exhaustive patterns: `Blue` not covered
    }
}
```

Fix: add the missing arm or use `_`:

```rust
fn describe(c: Color) -> &'static str {
    match c {
        Color::Red => "red",
        Color::Green => "green",
        Color::Blue => "blue",     // ✅ All variants covered
    }
}
```

---

## Pattern Types

### Literal Patterns

```rust
let x = 42;
match x {
    0 => println!("zero"),
    1 => println!("one"),
    42 => println!("the answer"),
    _ => println!("other"),
}

let c = 'a';
match c {
    'a'..='z' => println!("lowercase"),
    'A'..='Z' => println!("uppercase"),
    '0'..='9' => println!("digit"),
    _ => println!("other"),
}
```

### Variable Binding

A plain name in a pattern **binds** the matched value:

```rust
let x = 42;
match x {
    0 => println!("zero"),
    n => println!("got: {}", n),  // n binds the value 42
}
```

### Wildcard `_`

Matches anything but doesn't bind:

```rust
let pair = (1, true);
match pair {
    (0, _) => println!("first is zero"),
    (_, false) => println!("second is false"),
    _ => println!("anything else"),
}
```

### Multiple Patterns with `|`

```rust
let x = 3;
match x {
    1 | 2 => println!("one or two"),
    3 | 4 => println!("three or four"),
    _ => println!("other"),
}

// Works with enums too
match color {
    Color::Red | Color::Blue => println!("red or blue"),
    Color::Green => println!("green"),
}
```

### Range Patterns

```rust
let grade = 85;
match grade {
    90..=100 => println!("A"),
    80..=89 => println!("B"),
    70..=79 => println!("C"),
    60..=69 => println!("D"),
    0..=59 => println!("F"),
    _ => println!("Invalid grade"),
}
```

### Enum Patterns

```rust
enum Message {
    Quit,
    Echo(String),
    Move { x: i32, y: i32 },
    Color(u8, u8, u8),
}

fn process(msg: Message) {
    match msg {
        Message::Quit => println!("Quit"),
        Message::Echo(text) => println!("Echo: {}", text),
        Message::Move { x, y } => println!("Move to ({}, {})", x, y),
        Message::Color(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
    }
}
```

### Struct Patterns

```rust
struct Point {
    x: i32,
    y: i32,
}

fn classify(p: &Point) {
    match p {
        Point { x: 0, y: 0 } => println!("Origin"),
        Point { x, y: 0 } => println!("On x-axis at {}", x),
        Point { x: 0, y } => println!("On y-axis at {}", y),
        Point { x, y } => println!("({}, {})", x, y),
    }
}
```

### Tuple Patterns

```rust
let pair = (true, 42);
match pair {
    (true, n) if n > 0 => println!("true and positive: {}", n),
    (true, _) => println!("true"),
    (false, _) => println!("false"),
}
```

### Reference Patterns

When matching on references:

```rust
let values = vec![1, 2, 3];

for value in &values {
    match value {
        &1 => println!("one"),
        &2 => println!("two"),
        other => println!("other: {}", other),
    }
}
```

---

## Match Guards

Add conditions to arms with `if`:

```rust
let num = Some(4);
match num {
    Some(n) if n < 0 => println!("Negative: {}", n),
    Some(n) if n == 0 => println!("Zero"),
    Some(n) if n > 0 => println!("Positive: {}", n),
    Some(_) => unreachable!(),
    None => println!("Nothing"),
}
```

Guards are checked **after** the pattern matches:

```rust
let x = 4;
let even = true;

match x {
    n if even && n > 0 => println!("{} is positive and even flag is true", n),
    n => println!("{}", n),
}
```

---

## `@` Bindings

Bind a value while also testing it against a pattern:

```rust
let msg = Message::Echo(String::from("hello"));

match msg {
    Message::Echo(ref text @ _) if text.len() > 10 => println!("Long echo: {}", text),
    Message::Echo(text) => println!("Echo: {}", text),
    _ => {}
}

// More common with ranges:
let age = 25;
match age {
    n @ 0..=12 => println!("{} is a child", n),
    n @ 13..=19 => println!("{} is a teenager", n),
    n @ 20..=64 => println!("{} is an adult", n),
    n @ 65.. => println!("{} is a senior", n),
    _ => unreachable!(),
}
```

---

## Nested Matching

```rust
enum Expr {
    Num(i32),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
}

fn simplify(expr: &Expr) -> String {
    match expr {
        Expr::Num(n) => n.to_string(),
        Expr::Add(left, right) => {
            match (left.as_ref(), right.as_ref()) {
                (Expr::Num(0), other) | (other, Expr::Num(0)) => simplify(other),
                _ => format!("({} + {})", simplify(left), simplify(right)),
            }
        }
        Expr::Mul(left, right) => {
            match (left.as_ref(), right.as_ref()) {
                (Expr::Num(0), _) | (_, Expr::Num(0)) => "0".to_string(),
                (Expr::Num(1), other) | (other, Expr::Num(1)) => simplify(other),
                _ => format!("({} * {})", simplify(left), simplify(right)),
            }
        }
    }
}
```

---

## Common Patterns

### Matching on Multiple Values

```rust
let (x, y) = (true, false);
match (x, y) {
    (true, true) => println!("both true"),
    (true, false) => println!("x true, y false"),
    (false, true) => println!("x false, y true"),
    (false, false) => println!("both false"),
}
```

### Matching on Result and Option

```rust
fn process(input: &str) -> String {
    match input.parse::<i32>() {
        Ok(n) if n > 0 => format!("Positive: {}", n),
        Ok(n) if n < 0 => format!("Negative: {}", n),
        Ok(_) => "Zero".to_string(),
        Err(e) => format!("Error: {}", e),
    }
}
```

### Using matches! Macro

For simple boolean checks:

```rust
let x = 5;
let is_small = matches!(x, 1..=5);
println!("{}", is_small);  // true

let opt: Option<i32> = Some(42);
let has_value = matches!(opt, Some(_));

let color = Color::Red;
let is_primary = matches!(color, Color::Red | Color::Blue);
```

---

## Exercises

### Exercise 1: Grade Classifier

Write a function that takes a score (0-100) and returns a grade letter:

```rust
fn grade(score: u32) -> &'static str {
    // Use match with ranges
    // 90-100: "A", 80-89: "B", 70-79: "C", 60-69: "D", 0-59: "F"
}
```

<details>
<summary>Solution</summary>

```rust
fn grade(score: u32) -> &'static str {
    match score {
        90..=100 => "A",
        80..=89 => "B",
        70..=79 => "C",
        60..=69 => "D",
        0..=59 => "F",
        _ => "Invalid",
    }
}
```

</details>

### Exercise 2: Fizzbuzz with match

Rewrite fizzbuzz using match on a tuple:

```rust
for i in 1..=15 {
    let result = match (i % 3, i % 5) {
        // Your patterns here
    };
    println!("{}", result);
}
```

<details>
<summary>Solution</summary>

```rust
for i in 1..=15 {
    let result = match (i % 3, i % 5) {
        (0, 0) => String::from("FizzBuzz"),
        (0, _) => String::from("Fizz"),
        (_, 0) => String::from("Buzz"),
        _ => i.to_string(),
    };
    println!("{}", result);
}
```

</details>

---

## Summary

| Pattern | Example | Matches |
|---------|---------|---------|
| Literal | `42` | Exactly 42 |
| Variable | `n` | Anything, binds to n |
| Wildcard | `_` | Anything, no binding |
| Multiple | `1 \| 2 \| 3` | 1 or 2 or 3 |
| Range | `0..=100` | 0 through 100 inclusive |
| Enum | `Some(n)` | Some variant, binds inner |
| Struct | `Point { x, y: 0 }` | Point where y is 0 |
| Tuple | `(a, b)` | Any 2-tuple |
| Guard | `n if n > 0` | If condition is true |
| @ Binding | `n @ 1..=5` | Bind AND test range |

**Key Takeaways:**
1. `match` is an expression — it returns a value
2. It's **exhaustive** — every possible value must be handled
3. Arms are checked top to bottom — first match wins
4. Match guards add conditions with `if`
5. `@` binds a name while testing a pattern
6. `matches!` is a convenient macro for boolean pattern checks

---

**Next:** [if let & while let →](./07-if-let-while-let.md)

<p align="center"><i>Tutorial 6 of 8 — Stage 4: Structuring Data</i></p>

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

## Pattern Matching Across Programming Languages

Rust's `match` didn't appear out of nowhere. It descends from decades of programming language research — and understanding where it came from reveals why it's so much better than what most languages offer.

### The Evolution of Branching on Values

**C/C++ `switch` (1972):**
The original. Only matches integers and characters. No destructuring. Fall-through is the *default* behavior — forgetting a `break` is one of the most common bugs in C code. Decades of CVEs trace back to missing `break` statements.

**Haskell (1990):**
Pattern matching has been a core feature since day one. Haskell directly inspired Rust's design — algebraic data types + exhaustive pattern matching. If you know Haskell, Rust's `match` feels like home.

**OCaml / F# (1996 / 2005):**
Pattern matching on algebraic data types with exhaustiveness checking. OCaml was a *major* influence on Rust's early design (Rust's original compiler was written in OCaml!).

**Scala (2004):**
`match` with case classes — very similar to Rust. Scala proved these ideas worked on the JVM and brought ML-style matching to a wider audience.

**Erlang / Elixir:**
Pattern matching is THE primary control flow mechanism. Even function parameters use patterns. `receive` blocks in Erlang pattern-match on messages — the entire concurrency model is built on it.

**Swift (2014):**
`switch` with associated values and exhaustiveness. Apple was clearly paying attention to the ML family and Rust.

**Kotlin (2016):**
`when` expression with smart casts. Essentially pattern matching dressed in Java-friendly syntax.

**Java (2020+):**
Switch expressions in Java 14+ now support pattern matching for `instanceof` — roughly 20 years after Scala showed the JVM could do it.

**Python (2021):**
The `match` statement added in Python 3.10 — directly inspired by Rust and Scala. The PEP explicitly cites Rust as a design influence.

### Comparison Table

| Language | Keyword | Exhaustive? | Destructuring? | Expression? | Fall-through? |
|-----------|---------|-------------|----------------|-------------|---------------|
| C/C++ | `switch` | No | No | No | Yes (bug-prone) |
| Java 14+ | `switch` | Optional | Limited | Yes | No (new form) |
| Python 3.10+ | `match` | No | Yes | No | No |
| Haskell | `case` | Yes (warn) | Yes | Yes | No |
| OCaml | `match` | Yes | Yes | Yes | No |
| Scala | `match` | Yes (warn) | Yes | Yes | No |
| Swift | `switch` | Yes | Yes | No (stmt) | No |
| Kotlin | `when` | Yes (sealed) | Limited | Yes | No |
| Elixir | `case` | Yes (warn) | Yes | Yes | No |
| **Rust** | **`match`** | **Yes (error)** | **Yes** | **Yes** | **No** |

### Why Rust's Position Matters

```
 Academic Languages          Mainstream Languages
 ┌──────────────────┐        ┌──────────────────┐
 │ Haskell (1990)   │        │ C switch (1972)  │
 │ ML/OCaml (1996)  │───────>│ Java switch      │
 │ SML, F#          │        │ Python match     │
 └────────┬─────────┘        └──────────────────┘
          │                           ▲
          │    ┌──────────────────┐    │
          └───>│   Rust match     │────┘
               │ (systems + safe  │
               │  + exhaustive)   │
               └──────────────────┘
```

Rust was the first **systems programming language** to make exhaustive pattern matching on algebraic data types a core feature. It brought what was long considered an "academic" concept into a language designed for operating systems, embedded devices, and performance-critical code. Every language that adds pattern matching after Rust is, in some way, validating this design choice.

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

## How `match` Compiles — Decision Trees and Jump Tables

You might worry that all this safety comes at a runtime cost. It doesn't. In fact, `match` often compiles to **faster** code than hand-written `if/else` chains.

### The Compiler Doesn't Generate if/else

When the compiler sees a `match`, it doesn't naively emit a chain of comparisons. Instead, it builds a **decision tree** — an optimized structure that determines the minimum number of tests needed to find the matching arm.

```
 Source Code                    Compiled Decision Tree
 ┌─────────────────────┐        ┌──────────────────────────┐
 │ match x {           │        │ Load discriminant of x   │
 │   0 => expr_a,      │───────>│        │                 │
 │   1 => expr_b,      │        │   ┌────┴────┐            │
 │   2 => expr_c,      │        │ jump_table[0,1,2]        │
 │   _ => expr_d,      │        │   │    │    │   else     │
 │ }                   │        │   v    v    v    v       │
 └─────────────────────┘        │  ea   eb   ec   ed      │
                                └──────────────────────────┘
```

### Integer Patterns → Jump Tables

For contiguous integer ranges like `0, 1, 2, 3`, the compiler (via LLVM) emits a **jump table** — an array of code addresses indexed by the value. This is O(1) lookup, identical to what an optimized C `switch` produces:

```
 // Pseudo-assembly for match x { 0 => ..., 1 => ..., 2 => ..., _ => ... }
 cmp x, 2              // is x in range [0, 2]?
 ja  default_label      // if above, jump to default
 jmp [jump_table + x*8] // O(1) direct jump!
```

No chain of comparisons. No branch mispredictions from nested if/else. Just a single indexed jump.

### Enum Patterns → Discriminant Comparison

Every Rust enum has a hidden integer **discriminant** (think of it as a tag). Matching on enum variants compiles to comparing this discriminant, then jumping directly to the correct arm:

```rust
// This match:
match msg {
    Message::Quit => { /* ... */ }
    Message::Echo(text) => { /* ... */ }
    Message::Move { x, y } => { /* ... */ }
}
// Compiles roughly to:
// load discriminant (0=Quit, 1=Echo, 2=Move)
// jump table on discriminant value
// destructure fields only in the arm that needs them
```

### Exhaustiveness Checking — The Maranget Algorithm

The exhaustiveness checker is based on research by Luc Maranget (2008) — *"Compiling Pattern Matching to Good Decision Trees."* The algorithm works by:

1. **Building a pattern matrix** — each row is a match arm, each column is a position in the pattern
2. **Usefulness checking** — the compiler asks: "Would adding a wildcard `_` arm at the bottom be *useful*?" If no, every possible value is already covered
3. **Reachability checking** — if an arm can never be reached (shadowed by earlier arms), the compiler warns about it

This is the same algorithm that checks for **unreachable patterns**:

```rust
match x {
    0 => println!("zero"),
    n => println!("other: {}", n),
    42 => println!("never reached!"),  // ⚠️ Warning: unreachable pattern
}
```

### Performance Guarantee

| Technique | When Used | Complexity |
|-----------|-----------|------------|
| Jump table | Contiguous integers | O(1) |
| Binary search | Sparse integers | O(log n) |
| Discriminant switch | Enum variants | O(1) |
| Decision tree | Complex nested patterns | Optimal tests |

The key insight: **you get safety (exhaustiveness) AND performance — there is no tradeoff.** The compiler's pattern analysis produces code that is never slower than hand-written `if/else` chains, and is often faster because the optimizer has more information about the structure of the patterns.

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

## Advanced Pattern Matching Techniques

Before heading to the exercises, let's cover some deeper aspects of pattern matching that make Rust's system uniquely powerful.

### Patterns Are Everywhere

Rust uses patterns in far more places than just `match`. Anywhere you bind a variable, a pattern is involved:

```rust
// let bindings are patterns!
let (x, y, z) = (1, 2, 3);
let Point { x, y } = point;

// Function parameters are patterns!
fn distance(&Point { x, y }: &Point) -> f64 {
    ((x * x + y * y) as f64).sqrt()
}

// for loops use patterns!
for (index, value) in vec.iter().enumerate() { /* ... */ }

// if let and while let
if let Some(inner) = optional { /* ... */ }
while let Some(item) = stack.pop() { /* ... */ }
```

### Refutable vs. Irrefutable Patterns

This distinction is critical for understanding where each pattern can appear:

```
 ┌─────────────────────────────────────────────────────────────┐
 │                    Pattern Types                            │
 ├─────────────────────────┬───────────────────────────────────┤
 │     Irrefutable          │         Refutable                │
 │  (always matches)        │   (might not match)              │
 ├─────────────────────────┼───────────────────────────────────┤
 │  let x = value;          │  if let Some(x) = opt { }       │
 │  let (a, b) = tuple;     │  while let Ok(v) = iter.next()  │
 │  fn foo(x: i32) { }      │  match val { Pat => ... }       │
 │  for x in iter { }       │  let Some(x) = opt else { };    │
 ├─────────────────────────┼───────────────────────────────────┤
 │  Used in: let, fn args,  │  Used in: match, if let,        │
 │  for loops               │  while let, let-else            │
 └─────────────────────────┴───────────────────────────────────┘
```

If you try to use a refutable pattern where an irrefutable one is required, the compiler will tell you:

```rust
// ❌ ERROR: refutable pattern in local binding
let Some(x) = some_option;

// ✅ Use if let or let-else instead
let Some(x) = some_option else { return; };
if let Some(x) = some_option { /* use x */ }
```

### Pattern Binding Modes (RFC 2005)

Before Rust 1.26, matching on references was verbose:

```rust
// Old style — needed explicit `ref`:
match &some_option {
    &Some(ref inner) => println!("{}", inner),
    &None => println!("nothing"),
}

// New style — binding modes are automatic:
match &some_option {
    Some(inner) => println!("{}", inner),  // inner is &T automatically
    None => println!("nothing"),
}
```

The compiler automatically inserts `ref` when you match on a reference. This was added via RFC 2005 ("match ergonomics") and dramatically reduced the need to write `ref` and `&` in patterns. It's one of the reasons modern Rust code looks so clean.

### The `ref` Keyword — Mostly Historical

```rust
let value = String::from("hello");

// These are equivalent:
let ref r1 = value;      // r1: &String
let r2 = &value;          // r2: &String

// In older match code you'd see:
match value {
    ref s => println!("{}", s),  // borrows instead of moving
}
// Modern Rust rarely needs explicit ref thanks to binding modes
```

### The Power of Exhaustiveness in Practice

This is arguably the most important software engineering benefit of `match`. When you add a new variant to an enum:

```rust
enum Command {
    Start,
    Stop,
    Restart,  // ← newly added
}
```

The compiler immediately flags **every** `match` on `Command` that doesn't handle `Restart`. In a large codebase, this means:

- Zero chance of forgetting to handle the new case
- No need to grep through code hoping you found every usage
- Refactoring enums is fearless — the compiler is your checklist

This is why experienced Rust developers prefer explicit matching over `_ => ...` wildcards when possible — wildcards silently swallow new variants, defeating the safety net.

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

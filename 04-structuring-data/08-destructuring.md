# Destructuring 🧩

> **Destructuring lets you break apart structs, enums, tuples, and references into their individual components. Combined with pattern matching, it makes accessing nested data elegant and concise.**

---

## Table of Contents

- [What Is Destructuring?](#what-is-destructuring)
- [Destructuring Tuples](#destructuring-tuples)
- [Destructuring Structs](#destructuring-structs)
  - [Renaming Fields](#renaming-fields)
  - [Ignoring Fields](#ignoring-fields)
- [Destructuring Enums](#destructuring-enums)
- [Nested Destructuring](#nested-destructuring)
- [Destructuring in Function Parameters](#destructuring-in-function-parameters)
- [Destructuring References](#destructuring-references)
- [Destructuring in for Loops](#destructuring-in-for-loops)
- [Mixed Patterns](#mixed-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is Destructuring?

Destructuring is the opposite of constructing. You take a complex value apart into its pieces:

```rust
// Constructing
let point = (3, 4);

// Destructuring
let (x, y) = point;
println!("x = {}, y = {}", x, y);  // x = 3, y = 4
```

### Destructuring Across Programming Languages

Destructuring isn't unique to Rust — it's a concept that originated in functional languages and has
spread across the programming world. Understanding how other languages handle it helps you appreciate
what Rust brings to the table.

**JavaScript (ES6, 2015)** — probably the most widely known destructuring syntax:

```javascript
const { name, age } = person;          // Object destructuring
const [first, ...rest] = arr;          // Array destructuring with rest
const { a: { b: { c } } } = nested;   // Nested
```

**Python** — tuple unpacking has existed since Python 2, but there's no struct/dict destructuring:

```python
x, y, z = point            # Tuple unpacking
first, *rest = [1, 2, 3]   # Extended unpacking (Python 3)
# No equivalent of: { name, age } = person_dict
```

**Haskell** — pattern matching IS destructuring. They are the same concept:

```haskell
case point of (x, y) -> x + y      -- Tuple
let (Just val) = maybeValue          -- "Enum" (Maybe)
```

**Kotlin** — requires `componentN()` functions (destructuring declarations):

```kotlin
val (name, age) = person   // Calls person.component1(), person.component2()
val (key, value) = pair
```

**Swift** — tuple decomposition plus enum associated value matching:

```swift
let (x, y) = point
case .success(let data) = result   // Enum pattern
```

**Go** — a limited form: multiple return value unpacking only:

```go
x, err := someFunction()   // That's it — no struct or slice destructuring
```

**C++17** — structured bindings, added 45 years after the birth of C:

```cpp
auto [x, y, z] = std::make_tuple(1, 2, 3);
auto [key, value] = *map.begin();
```

**The historical pattern:**

```
1980s-90s  Functional languages (ML, Haskell) — pattern matching is core
2009       Clojure — first-class destructuring in a Lisp
2010       Rust (pre-1.0) — deep pattern matching from day one
2015       JavaScript ES6 — destructuring goes mainstream
2016       Kotlin 1.0 — componentN() declarations
2017       C++17 — structured bindings
```

> Destructuring started in functional languages and has been adopted by mainstream languages
> over the past decade. Rust's version is deeply integrated with its pattern matching
> system — more powerful than JavaScript's but built on the same fundamental concept.

| Language   | Tuples | Structs/Objects | Enums/Unions | Nested | In Fn Params |
|------------|--------|-----------------|--------------|--------|--------------|
| Rust       | ✅     | ✅              | ✅           | ✅     | ✅           |
| JavaScript | ✅     | ✅              | ❌           | ✅     | ✅           |
| Python     | ✅     | ❌              | ❌           | ✅     | ❌           |
| Haskell    | ✅     | ✅              | ✅           | ✅     | ✅           |
| Kotlin     | ✅     | ✅              | ✅           | ❌     | ❌           |
| Swift      | ✅     | ❌              | ✅           | ✅     | ❌           |
| Go         | ❌     | ❌              | ❌           | ❌     | ❌           |
| C++17      | ✅     | ✅              | ❌           | ❌     | ❌           |

---

## Destructuring Tuples

```rust
fn main() {
    let tuple = (1, "hello", 3.14);
    let (a, b, c) = tuple;
    println!("{} {} {}", a, b, c);
    
    // Ignore parts
    let (first, _, _) = tuple;
    let (_, second, _) = tuple;
    
    // With ..
    let (first, ..) = tuple;  // Ignore everything after first
    let (.., last) = tuple;   // Ignore everything before last
    
    // Nested tuples
    let nested = ((1, 2), (3, 4));
    let ((a, b), (c, d)) = nested;
    println!("{} {} {} {}", a, b, c, d);
}
```

---

## Destructuring Structs

```rust
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    
    // Destructure into variables matching field names
    let Point { x, y } = p;
    println!("x = {}, y = {}", x, y);
}
```

### Renaming Fields

```rust
let p = Point { x: 3.0, y: 4.0 };
let Point { x: horizontal, y: vertical } = p;
println!("{} {}", horizontal, vertical);
```

### Ignoring Fields

```rust
struct User {
    name: String,
    email: String,
    age: u32,
    active: bool,
}

fn main() {
    let user = User {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        age: 30,
        active: true,
    };
    
    // Only extract what you need
    let User { name, age, .. } = &user;
    println!("{} is {} years old", name, age);
}
```

In `match`:

```rust
fn classify(p: &Point) -> &str {
    match p {
        Point { x: 0.0, y: 0.0 } => "origin",
        Point { x: 0.0, .. } => "on y-axis",
        Point { y: 0.0, .. } => "on x-axis",
        _ => "elsewhere",
    }
}
```

---

## Destructuring Enums

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

fn describe(shape: &Shape) {
    match shape {
        Shape::Circle { radius } => {
            println!("Circle with radius {}", radius);
        }
        Shape::Rectangle { width, height } => {
            println!("{}x{} rectangle", width, height);
        }
        Shape::Triangle { base, height } => {
            println!("Triangle: base={}, height={}", base, height);
        }
    }
}
```

Nested enum destructuring:

```rust
fn describe_option_num(val: Option<i32>) {
    match val {
        Some(n) if n > 0 => println!("Positive: {}", n),
        Some(0) => println!("Zero"),
        Some(n) => println!("Negative: {}", n),
        None => println!("Nothing"),
    }
}
```

---

## Nested Destructuring

```rust
struct Address {
    city: String,
    zip: String,
}

struct Person {
    name: String,
    address: Address,
}

fn main() {
    let person = Person {
        name: String::from("Alice"),
        address: Address {
            city: String::from("Rustville"),
            zip: String::from("12345"),
        },
    };
    
    // Destructure nested structs
    let Person {
        name,
        address: Address { city, zip },
    } = &person;
    
    println!("{} lives in {} ({})", name, city, zip);
}
```

Nested with enums:

```rust
enum Payment {
    Cash(f64),
    Card { number: String, amount: f64 },
    Transfer { from: String, to: String, amount: f64 },
}

fn process(payment: &Payment) {
    match payment {
        Payment::Cash(amount) => println!("Cash: ${:.2}", amount),
        Payment::Card { amount, .. } => println!("Card: ${:.2}", amount),
        Payment::Transfer { from, to, amount } => {
            println!("Transfer ${:.2} from {} to {}", amount, from, to);
        }
    }
}
```

---

## Destructuring in Function Parameters

```rust
struct Point { x: f64, y: f64 }

fn distance_from_origin(&Point { x, y }: &Point) -> f64 {
    (x * x + y * y).sqrt()
}

fn print_pair((a, b): (i32, i32)) {
    println!("{} and {}", a, b);
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("Distance: {}", distance_from_origin(&p));  // 5.0
    
    print_pair((10, 20));  // "10 and 20"
}
```

---

## Destructuring References

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    
    // When iterating references, destructure to get the inner value
    for &num in &numbers {
        println!("{}", num);  // num is i32, not &i32
    }
    
    // Equivalent:
    for num in &numbers {
        println!("{}", *num);  // Dereference manually
    }
    
    // With tuples
    let pairs = vec![(1, 'a'), (2, 'b'), (3, 'c')];
    for &(num, letter) in &pairs {
        println!("{}: {}", num, letter);
    }
}
```

### Destructuring and Ownership — Rust's Unique Twist

In most languages, destructuring is a **purely syntactic convenience** — you extract values and move on.
In Rust, destructuring interacts with the **ownership system**, and that changes everything.

**Moving fields out of a struct:**

When you destructure a struct by value, you **move** each field out (unless the field is `Copy`):

```rust
struct Player {
    name: String,   // Not Copy — will be moved
    score: u32,     // Copy — will be copied
}

let player = Player { name: String::from("Alice"), score: 42 };
let Player { name, score } = player;

// `name` now owns the String, `score` is a copy of 42
println!("{}: {}", name, score);  // ✅ Works
// println!("{}", player.name);   // ❌ Error: player.name was moved
// println!("{}", player.score);  // ❌ Error: player is partially moved
```

After partial destructuring, the struct is **partially moved** — you can't use it as a whole.

**Using `ref` to borrow instead of move:**

```rust
let player = Player { name: String::from("Alice"), score: 42 };
let Player { ref name, score } = player;

// `name` is &String (borrowed), `score` is u32 (copied)
println!("{}: {}", name, score);  // ✅
println!("{}", player.name);      // ✅ Still valid — we only borrowed
```

**Pattern binding modes (RFC 2005) — automatic borrowing:**

```rust
let player = Player { name: String::from("Alice"), score: 42 };
let Player { name, score } = &player;  // Note the & on the right side

// Rust sees `&player` and automatically binds `name` as &String
// This is equivalent to writing `ref name` explicitly
println!("{}: {}", name, score);  // ✅
println!("{}", player.name);      // ✅ player is still fully intact
```

**How this differs from JavaScript:**

```
    JavaScript                          Rust
    ──────────────────────────          ──────────────────────────
    const { name } = person;            let Player { name, .. } = player;
    // `name` is a reference              // `name` MOVES the String out
    // to the same string object          // `player.name` is now invalid
    person.name === name  // true         // No equivalent — value moved

    // JS always copies primitives        // Rust copies only Copy types
    // and shares references              // and moves everything else
```

**The `..` pattern and safety:**

```rust
struct Config {
    name: String,
    debug: bool,
    retries: u32,
    timeout_ms: u64,
}

let config = Config { name: "app".into(), debug: true, retries: 3, timeout_ms: 5000 };
let Config { name, .. } = config;
// `name` was moved out, `debug`, `retries`, `timeout_ms` are dropped
// Rust tracks exactly which fields were moved and which weren't
```

> **Key insight:** In Rust, destructuring is not just syntax — it's a **transfer of ownership**.
> The compiler tracks every field individually, ensuring you never use moved data.
> This is what makes Rust's destructuring both more powerful and more disciplined than
> any garbage-collected language.

---

## Destructuring in `for` Loops

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert("Alice", 95);
    scores.insert("Bob", 87);
    
    // Destructure each (key, value) pair
    for (name, score) in &scores {
        println!("{}: {}", name, score);
    }
    
    // With enumerate
    let colors = vec!["red", "green", "blue"];
    for (index, color) in colors.iter().enumerate() {
        println!("{}: {}", index, color);
    }
    
    // Nested destructuring in loop
    let data = vec![
        ("Alice", vec![90, 85, 92]),
        ("Bob", vec![78, 88, 95]),
    ];
    
    for (name, scores) in &data {
        let avg: f64 = scores.iter().sum::<i32>() as f64 / scores.len() as f64;
        println!("{}: avg {:.1}", name, avg);
    }
}
```

---

## Mixed Patterns

Combine everything:

```rust
enum Command {
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
    Quit,
}

fn handle(commands: &[Command]) {
    for cmd in commands {
        match cmd {
            Command::Move { x: 0, y } => println!("Move vertically to y={}", y),
            Command::Move { x, y: 0 } => println!("Move horizontally to x={}", x),
            Command::Move { x, y } => println!("Move to ({}, {})", x, y),
            Command::Write(text) if text.is_empty() => println!("Empty write"),
            Command::Write(text) => println!("Write: {}", text),
            Command::ChangeColor(r, g, b) => println!("Color: #{:02x}{:02x}{:02x}", r, g, b),
            Command::Quit => println!("Quit!"),
        }
    }
}
```

### Real-World Destructuring Patterns

Beyond the basics, destructuring becomes a daily tool in idiomatic Rust. Here are the
patterns experienced Rust developers use constantly.

**Pattern 1: Destructure directly in function signatures**

```rust
// Instead of accessing fields inside the body...
fn distance(p1: &(f64, f64), p2: &(f64, f64)) -> f64 {
    ((p1.0 - p2.0).powi(2) + (p1.1 - p2.1).powi(2)).sqrt()
}

// ...destructure in the signature itself:
fn distance(&(x1, y1): &(f64, f64), &(x2, y2): &(f64, f64)) -> f64 {
    ((x1 - x2).powi(2) + (y1 - y2).powi(2)).sqrt()
}
```

**Pattern 2: Match + destructure in one step**

```rust
enum Event {
    Click { x: i32, y: i32, button: u8 },
    Scroll { delta: f64 },
    KeyPress(char),
}

fn handle(event: &Event) {
    match event {
        Event::Click { x, y, button } => println!("Click btn {} at ({},{})", button, x, y),
        Event::Scroll { delta } if *delta > 0.0 => println!("Scroll up {}", delta),
        Event::Scroll { delta } => println!("Scroll down {}", delta),
        Event::KeyPress(c) => println!("Key: {}", c),
    }
}
```

**Pattern 3: Arbitrarily deep nested destructuring**

```rust
struct Point { x: f64, y: f64 }

let pair = (10, 20);
let point = Point { x: 3.0, y: 4.0 };
let ((a, b), Point { x, y }) = (pair, point);
// a=10, b=20, x=3.0, y=4.0 — all extracted in one statement
```

**Pattern 4: Destructuring in `for` loops — extremely common**

```rust
let names = vec!["Alice", "Bob", "Charlie"];

// enumerate gives (index, value) tuples — destructure them:
for (i, name) in names.iter().enumerate() {
    println!("#{}: {}", i + 1, name);
}

// zip two iterators and destructure:
let scores = vec![95, 87, 91];
for (name, score) in names.iter().zip(scores.iter()) {
    println!("{}: {}", name, score);
}
```

**Pattern 5: `if let` chains with destructuring**

```rust
struct Point { x: f64, y: f64 }

let maybe_point: Option<Point> = Some(Point { x: 1.0, y: 2.0 });

if let Some(Point { x, y }) = &maybe_point {
    println!("Got point at ({}, {})", x, y);
}
```

**Advanced: `@` bindings — destructure AND keep the whole value**

```rust
#[derive(Debug)]
struct Point { x: i32, y: i32 }

let point = Point { x: 5, y: 12 };

match point {
    p @ Point { x: 0..=10, y: 0..=10 } => {
        println!("{:?} is in the 10x10 box", p);  // Use the whole struct
    }
    p @ Point { x, y } => {
        println!("{:?} is outside at ({}, {})", p, x, y);  // Use both
    }
}
```

> The `@` binding says "match this pattern, but also bind the entire value to a name."
> This lets you inspect the structure AND keep a handle to the whole thing.

**Anti-pattern: Over-destructuring**

```rust
// ❌ Over-destructured — harder to read when you only need one field:
let Point { x, y: _ } = &point;
println!("x = {}", x);

// ✅ Just use dot notation:
println!("x = {}", point.x);
```

**Rule of thumb:** Destructure when you use **multiple fields** in a computation.
Use dot notation when accessing **one or two fields**.

> **Connection to functional programming:** Destructuring is a form of *structural
> pattern matching* — your code "sees through" the shape of data and extracts
> exactly what it needs. This is why languages with strong pattern matching
> (Haskell, OCaml, Rust) tend to produce concise, readable code.

---

## Exercises

### Exercise 1: Destructure a Nested Struct

Given these types, write a function that extracts the city name:

```rust
struct Address { street: String, city: String }
struct Company { name: String, hq: Address }

fn hq_city(company: &Company) -> &str {
    // Use destructuring
}
```

<details>
<summary>Solution</summary>

```rust
fn hq_city(company: &Company) -> &str {
    let Company { hq: Address { city, .. }, .. } = company;
    city
}
```

</details>

### Exercise 2: Destructure Tuples

Write a function that takes a slice of `(name, score)` tuples and returns the name with the highest score:

```rust
fn top_scorer<'a>(scores: &'a [(&str, u32)]) -> Option<&'a str> {
    // Your code using destructuring
}
```

<details>
<summary>Solution</summary>

```rust
fn top_scorer<'a>(scores: &'a [(&str, u32)]) -> Option<&'a str> {
    let mut best: Option<(&str, u32)> = None;
    for &(name, score) in scores {
        match best {
            None => best = Some((name, score)),
            Some((_, best_score)) if score > best_score => best = Some((name, score)),
            _ => {}
        }
    }
    best.map(|(name, _)| name)
}
```

</details>

---

## Summary

| What | Pattern | Example |
|------|---------|---------|
| Tuple | `(a, b, c)` | `let (x, y) = (1, 2);` |
| Struct | `Type { field, .. }` | `let Point { x, y } = p;` |
| Enum | `Variant(data)` | `Some(val) => ...` |
| Nested | Mix of above | `Person { address: Address { city, .. }, .. }` |
| Ignore | `_` or `..` | `(first, ..)` |
| Rename | `field: name` | `Point { x: horizontal, .. }` |
| Reference | `&pattern` | `for &num in &vec` |

**Key Takeaways:**
1. Destructuring breaks complex values into parts
2. Works in `let`, `match`, `if let`, function parameters, and `for` loops
3. Use `..` to ignore remaining fields
4. Use `_` to ignore a single element
5. Nesting destructuring works to any depth
6. Patterns can combine destructuring with guards and `@` bindings

---

**Next Stage:** [Stage 5: Collections →](../05-collections/README.md)

<p align="center"><i>Tutorial 8 of 8 — Stage 4: Structuring Data</i></p>

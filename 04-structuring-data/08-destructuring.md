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

# Enums — Defining Variants 🎭

> **Enums in Rust are far more powerful than in most languages. Each variant can hold different types and amounts of data. Combined with pattern matching, they let you model complex states with perfect type safety.**

---

## Table of Contents

- [What Is an Enum?](#what-is-an-enum)
- [Simple Enums (C-style)](#simple-enums-c-style)
- [Enums with Data](#enums-with-data)
  - [Tuple Variants](#tuple-variants)
  - [Struct Variants](#struct-variants)
  - [Mixed Variants](#mixed-variants)
- [Enum Memory Layout](#enum-memory-layout)
- [Enums and impl](#enums-and-impl)
- [Enums as State Machines](#enums-as-state-machines)
- [Common Standard Library Enums](#common-standard-library-enums)
- [Enums vs Structs — When to Use Which](#enums-vs-structs--when-to-use-which)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is an Enum?

An **enum** (enumeration) defines a type that can be **one of several variants**. Where a struct says "AND" (has name AND age AND email), an enum says "OR" (is Red OR Green OR Blue):

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn main() {
    let d = Direction::North;
    
    match d {
        Direction::North => println!("Going up!"),
        Direction::South => println!("Going down!"),
        Direction::East  => println!("Going right!"),
        Direction::West  => println!("Going left!"),
    }
}
```

---

## Simple Enums (C-style)

Enums where variants have no data — just named choices:

```rust
#[derive(Debug, PartialEq)]
enum Color {
    Red,
    Green,
    Blue,
}

#[derive(Debug)]
enum DayOfWeek {
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday,
}

impl DayOfWeek {
    fn is_weekend(&self) -> bool {
        matches!(self, DayOfWeek::Saturday | DayOfWeek::Sunday)
    }
}

fn main() {
    let today = DayOfWeek::Saturday;
    println!("{:?} is weekend: {}", today, today.is_weekend());
}
```

You can assign integer values (discriminants):

```rust
#[derive(Debug)]
enum HttpStatus {
    Ok = 200,
    NotFound = 404,
    InternalServerError = 500,
}

fn main() {
    println!("{}", HttpStatus::Ok as i32);           // 200
    println!("{}", HttpStatus::NotFound as i32);      // 404
}
```

---

## Enums with Data

This is where Rust enums shine. Each variant can hold different data. This is sometimes called a **sum type** or **tagged union**.

### Tuple Variants

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    let home = IpAddr::V4(127, 0, 0, 1);
    let loopback = IpAddr::V6(String::from("::1"));
    
    match home {
        IpAddr::V4(a, b, c, d) => println!("{}.{}.{}.{}", a, b, c, d),
        IpAddr::V6(addr) => println!("{}", addr),
    }
}
```

### Struct Variants

Variants with named fields:

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        Shape::Rectangle { width, height } => width * height,
        Shape::Triangle { base, height } => 0.5 * base * height,
    }
}

fn main() {
    let shapes = vec![
        Shape::Circle { radius: 5.0 },
        Shape::Rectangle { width: 10.0, height: 3.0 },
        Shape::Triangle { base: 8.0, height: 4.0 },
    ];
    
    for shape in &shapes {
        println!("Area: {:.2}", area(shape));
    }
}
```

### Mixed Variants

Different variants can have different kinds of data — or no data at all:

```rust
enum Command {
    Quit,                              // No data
    Echo(String),                      // Tuple variant
    Move { x: i32, y: i32 },          // Struct variant
    ChangeColor(u8, u8, u8),           // Tuple variant
}

fn process(cmd: Command) {
    match cmd {
        Command::Quit => println!("Quitting"),
        Command::Echo(msg) => println!("{}", msg),
        Command::Move { x, y } => println!("Moving to ({}, {})", x, y),
        Command::ChangeColor(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
    }
}
```

---

## Enum Memory Layout

An enum is as big as its **largest variant** plus a **discriminant** (tag):

```rust
use std::mem;

enum Example {
    A,                  // 0 bytes of data
    B(u8),              // 1 byte of data
    C(u64),             // 8 bytes of data
    D(String),          // 24 bytes of data (ptr + len + cap)
}

fn main() {
    println!("Size: {} bytes", mem::size_of::<Example>());
    // 32 bytes: 24 (largest variant: String) + 8 (discriminant + padding)
}
```

```
Memory layout (simplified):
┌──────────────┬─────────────────────────────┐
│ discriminant │ payload (size of largest)    │
│   (tag)      │                             │
└──────────────┴─────────────────────────────┘

Variant A: [tag=0][------- unused -------]
Variant B: [tag=1][u8][---- unused ------]
Variant C: [tag=2][  u64  ][-- unused ---]
Variant D: [tag=3][ ptr ][ len ][ cap ]
```

The discriminant tells Rust which variant is stored. The unused space is padding — it's allocated but not used by smaller variants.

---

## Enums and `impl`

Enums can have methods just like structs:

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

impl Coin {
    fn value_in_cents(&self) -> u32 {
        match self {
            Coin::Penny => 1,
            Coin::Nickel => 5,
            Coin::Dime => 10,
            Coin::Quarter => 25,
        }
    }
    
    fn name(&self) -> &str {
        match self {
            Coin::Penny => "penny",
            Coin::Nickel => "nickel",
            Coin::Dime => "dime",
            Coin::Quarter => "quarter",
        }
    }
}

fn main() {
    let coins = vec![Coin::Quarter, Coin::Dime, Coin::Penny, Coin::Penny];
    let total: u32 = coins.iter().map(|c| c.value_in_cents()).sum();
    println!("Total: {} cents", total);  // 37
}
```

---

## Enums as State Machines

Enums are perfect for modeling states:

```rust
enum OrderStatus {
    Pending,
    Processing { started_at: String },
    Shipped { tracking: String },
    Delivered { delivered_at: String },
    Cancelled { reason: String },
}

impl OrderStatus {
    fn description(&self) -> String {
        match self {
            OrderStatus::Pending => "Order is pending".to_string(),
            OrderStatus::Processing { started_at } => 
                format!("Processing since {}", started_at),
            OrderStatus::Shipped { tracking } => 
                format!("Shipped — tracking: {}", tracking),
            OrderStatus::Delivered { delivered_at } => 
                format!("Delivered on {}", delivered_at),
            OrderStatus::Cancelled { reason } => 
                format!("Cancelled: {}", reason),
        }
    }
    
    fn is_final(&self) -> bool {
        matches!(self, OrderStatus::Delivered { .. } | OrderStatus::Cancelled { .. })
    }
}
```

---

## Common Standard Library Enums

```rust
// Option<T> — value or nothing
enum Option<T> {
    Some(T),
    None,
}

// Result<T, E> — success or error
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// Ordering — comparison result
enum Ordering {
    Less,
    Equal,
    Greater,
}
```

We'll cover `Option` and `Result` in the next two tutorials in detail.

---

## Enums vs Structs — When to Use Which

```
Use a STRUCT when something has multiple pieces of data AT THE SAME TIME:
  → A user has a name AND an email AND an age
  → struct User { name, email, age }

Use an ENUM when something can be ONE OF SEVERAL VARIANTS:
  → A shape is a circle OR a rectangle OR a triangle
  → enum Shape { Circle, Rectangle, Triangle }
```

| Feature | Struct | Enum |
|---------|--------|------|
| Relationship | AND (all fields) | OR (one variant) |
| Data | All fields present | One variant's data |
| Pattern matching | Destructure fields | Match variants |
| Memory | Sum of fields | Size of largest variant |
| Adding variants | N/A | Adding a variant is easy |

---

## Exercises

### Exercise 1: Traffic Light

Create a `TrafficLight` enum with `Red`, `Yellow`, `Green` variants. Add a method `duration_seconds` that returns how long each light lasts (Red: 60, Yellow: 5, Green: 45) and a method `next` that returns the next light in the cycle.

<details>
<summary>Solution</summary>

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

impl TrafficLight {
    fn duration_seconds(&self) -> u32 {
        match self {
            TrafficLight::Red => 60,
            TrafficLight::Yellow => 5,
            TrafficLight::Green => 45,
        }
    }
    
    fn next(&self) -> TrafficLight {
        match self {
            TrafficLight::Red => TrafficLight::Green,
            TrafficLight::Yellow => TrafficLight::Red,
            TrafficLight::Green => TrafficLight::Yellow,
        }
    }
}
```

</details>

### Exercise 2: Arithmetic Expression

Define an enum that represents a math expression and evaluate it:

```rust
enum Expr {
    Num(f64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
}
```

Write `fn eval(expr: &Expr) -> f64`.

<details>
<summary>Solution</summary>

```rust
fn eval(expr: &Expr) -> f64 {
    match expr {
        Expr::Num(n) => *n,
        Expr::Add(left, right) => eval(left) + eval(right),
        Expr::Mul(left, right) => eval(left) * eval(right),
    }
}

fn main() {
    // (2 + 3) * 4
    let expr = Expr::Mul(
        Box::new(Expr::Add(
            Box::new(Expr::Num(2.0)),
            Box::new(Expr::Num(3.0)),
        )),
        Box::new(Expr::Num(4.0)),
    );
    println!("{}", eval(&expr));  // 20.0
}
```

</details>

---

## Summary

| Feature | Syntax |
|---------|--------|
| Simple variant | `enum E { A, B, C }` |
| Tuple variant | `enum E { A(T1, T2) }` |
| Struct variant | `enum E { A { field: T } }` |
| Create | `let x = E::A;` |
| With data | `let x = E::A(value);` |
| Methods | `impl E { fn method(&self) { match self { ... } } }` |

**Key Takeaways:**
1. Enum variants are different **possibilities** — the value is one of them
2. Each variant can hold different types and amounts of data
3. Enums are sized to their largest variant plus a discriminant
4. Methods on enums typically `match` on `self`
5. Enums are ideal for state machines, commands, and type-safe alternatives
6. `Option` and `Result` are the most important standard enums

---

**Next:** [The Option Enum →](./04-option-enum.md)

<p align="center"><i>Tutorial 3 of 8 — Stage 4: Structuring Data</i></p>

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

### The Evolution of Enums — From C to Rust

Rust's enums didn't appear out of nowhere. They represent decades of programming language evolution — from simple named integers to full algebraic data types. Understanding this history helps you appreciate *why* Rust enums work the way they do.

**C enums (1972)** were the simplest possible approach — just named integer constants:

```c
// C — enums are just integers in disguise
enum Color { RED = 0, GREEN = 1, BLUE = 2 };
enum Color c = 42;  // Compiles! No type safety at all.
```

A C enum provides zero guarantees. Any integer can be assigned to an enum variable, and variants cannot carry data.

**Java enums (2004)** improved things by making each variant a class instance:

```java
// Java — variants are singleton objects, but all share the same fields
enum Planet {
    EARTH(5.97e24, 6.37e6),
    MARS(6.42e23, 3.39e6);
    final double mass, radius;
    Planet(double m, double r) { mass = m; radius = r; }
}
```

Java enums are type-safe and can have methods, but every variant must have the **same fields** — you can't have one variant hold a `String` and another hold two `f64`s.

**Haskell (1990)** introduced **algebraic data types (ADTs)** — the academic ancestor of Rust's enums:

```haskell
-- Haskell — each variant (constructor) can hold different data
data Shape = Circle Double | Rectangle Double Double | Triangle Double Double
```

**ML family (OCaml, F#)** calls them **discriminated unions** or **variant types** — same concept, different name. **Swift** calls them enums with **associated values** — Swift and Rust influenced each other heavily. **TypeScript** achieves something similar with **discriminated unions** using type guards:

```typescript
// TypeScript — manual "tagged union" pattern
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number };
```

Rust took the algebraic data type concept from Haskell/ML and made it **mainstream** — no PhD required.

#### Sum Types vs Product Types

An enum is a **sum type** — a value is ONE of several variants (like *addition* of possibilities). A struct is a **product type** — a value contains ALL fields (like *multiplication* of possibilities):

```
Sum type (enum):     # of possible values = V1 + V2 + V3
Product type (struct): # of possible values = F1 × F2 × F3

enum Light { Red, Yellow, Green }       → 3 possible values (1 + 1 + 1)
struct Point { x: bool, y: bool }       → 4 possible values (2 × 2)
```

| Language | Enum Name | Can Hold Data? | Type-Safe? | Pattern Matchable? |
|------------|----------------------|----------------|------------|--------------------|
| C | `enum` | No | No | No |
| Java | `enum` | Yes (same shape) | Yes | Limited (switch) |
| Haskell | Algebraic data type | Yes (per-variant) | Yes | Yes |
| OCaml / F# | Discriminated union | Yes (per-variant) | Yes | Yes |
| Swift | `enum` (assoc. vals) | Yes (per-variant) | Yes | Yes |
| TypeScript | Discriminated union | Yes (per-variant) | Partial | Yes (type guards) |
| **Rust** | **`enum`** | **Yes (per-variant)** | **Yes** | **Yes (exhaustive)** |

Rust enums combine the best of all worlds: the familiarity of C-style syntax, the power of Haskell ADTs, and the compiler-enforced exhaustive matching that catches bugs at compile time.

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

### Deep Dive — The Discriminant and Niche Optimization

Every enum value contains a hidden **discriminant** (also called a **tag**) — a small integer that tells the runtime which variant is currently stored. The compiler automatically chooses the smallest sufficient integer type:

```
  ≤ 256 variants  →  u8  (1 byte discriminant)
  ≤ 65,536 variants  →  u16 (2 bytes)
  ... and so on
```

You can explicitly control the discriminant size with `#[repr]` — essential for **FFI** (calling C code):

```rust
#[repr(u32)]  // Force 4-byte discriminant for C compatibility
enum CStatus {
    Ok = 0,
    Error = 1,
    Timeout = 2,
}
```

**Size formula:** `size_of::<Enum>() = discriminant + LARGEST variant + padding`

Let's trace a concrete example:

```rust
enum Msg {
    Quit,                        // 0 bytes payload
    Echo(String),                // 24 bytes (ptr + len + cap)
    Move { x: i32, y: i32 },    // 8 bytes
}
// discriminant: 1 byte (only 3 variants)
// largest variant: 24 bytes (String)
// alignment: 8 bytes (String contains pointers)
// total: 32 bytes (1 + 24 = 25, rounded up to 32 for alignment)
```

```
┌──────────┬─────────────────────────────────────────────┐
│ tag (1B) │ padding (7B) │ payload area (24B)           │
├──────────┼─────────────────────────────────────────────┤
│ Quit   0 │ ---- empty --------------------------------│
│ Echo   1 │              │ ptr(8) │ len(8) │ cap(8)     │
│ Move   2 │              │  x(4)  │  y(4)  │ unused(16) │
└──────────┴─────────────────────────────────────────────┘
```

#### Niche Optimization — Zero-Cost Optionals

Here's where Rust does something **remarkable**. Consider `Option<&T>` — you'd expect it to be 16 bytes (8 for the reference + 8 for the discriminant with alignment). But it's only **8 bytes** — the same size as a bare `&T`!

How? A valid reference can **never** be null (address `0x0`). The compiler recognizes this "niche" — an invalid bit pattern — and reuses it as the `None` discriminant:

```
Option<&T> layout (8 bytes total — no extra discriminant!):
 ┌───────────────────────────────────┐
 │ Some(&T):  0x00007fff4a2b (valid) │  ← any non-zero address
 │ None:      0x000000000000 (null)  │  ← null means None
 └───────────────────────────────────┘
```

This is called **niche filling** — the compiler finds bit patterns that are invalid for the inner type and reuses them. More examples:

```rust
use std::mem::size_of;
use std::num::NonZeroU32;

assert_eq!(size_of::<Option<&i32>>(),     8);  // same as &i32!
assert_eq!(size_of::<Option<NonZeroU32>>(), 4);  // same as u32!
assert_eq!(size_of::<Option<bool>>(),     1);  // same as bool!
assert_eq!(size_of::<Option<Box<i32>>>(),  8);  // same as Box<i32>!
```

| Type | Size | `Option<T>` Size | Overhead |
|------|------|-------------------|----------|
| `&T` | 8 | 8 | **0 bytes** |
| `Box<T>` | 8 | 8 | **0 bytes** |
| `NonZeroU32` | 4 | 4 | **0 bytes** |
| `bool` | 1 | 1 | **0 bytes** |
| `u32` | 4 | 8 | 4 bytes |
| `String` | 24 | 32 | 8 bytes |

This optimization is **unique to Rust** among mainstream languages. In Java, every `Optional<T>` allocates a separate object on the heap. In C++, `std::optional<T>` always adds at least one byte. Rust's niche filling means you can use `Option` everywhere with literally zero runtime cost for pointer-like types.

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

### Enums as State Machines — A Design Pattern Deep Dive

State machines are **everywhere** in programming: HTTP request lifecycles, game entities, UI component states, protocol parsers, text editors — any time something transitions through well-defined stages. Let's examine why enums are the ideal tool for modeling them.

**The traditional OOP approach** uses the State design pattern — an interface with virtual methods and a different class for each state:

```
OOP State Pattern:                     Rust Enum Approach:
┌─────────────┐                        ┌──────────────────────────┐
│ «interface»  │                        │ enum ConnState {         │
│ State        │                        │   Listening,             │
│  +handle()   │                        │   Connected { addr },    │
├─────────────┤                        │   Transferring { bytes },│
│ Listening    │  ← heap-allocated      │   Closed { reason },     │
│ Connected    │  ← runtime dispatch    │ }                        │
│ Transferring │  ← hard to see         │                          │
│ Closed       │    all states          │ // All states visible in │
└─────────────┘                        │ // one place. Compiler   │
                                       │ // enforces exhaustive   │
                                       │ // handling.             │
                                       └──────────────────────────┘
```

Consider a TCP-like connection modeled as an enum:

```rust
enum TcpState {
    Listen,
    SynReceived { peer: String },
    Established { peer: String, bytes_sent: u64 },
    FinWait { remaining_acks: u32 },
    Closed,
}

impl TcpState {
    fn transition(self, event: &str) -> TcpState {
        match (self, event) {
            (TcpState::Listen, "syn") =>
                TcpState::SynReceived { peer: "client".into() },
            (TcpState::SynReceived { peer }, "ack") =>
                TcpState::Established { peer, bytes_sent: 0 },
            (TcpState::Established { peer, .. }, "fin") =>
                TcpState::FinWait { remaining_acks: 1 },
            (TcpState::FinWait { remaining_acks: 0 }, "timeout") =>
                TcpState::Closed,
            (state, _) => state,  // Invalid transition — stay in current state
        }
    }
}
```

#### Why This Beats String-Based States

In JavaScript or Python, states are often represented as strings:

```
Python:  order.status = "shiped"   ← typo compiles, bug at runtime
Rust:    OrderStatus::Shiped       ← compiler error: variant not found
```

This embodies the **"making illegal states unrepresentable"** philosophy: if your code compiles, it's impossible for the system to be in an undefined state. You cannot accidentally create an `OrderStatus` that doesn't exist, and you cannot forget to handle a variant (the match exhaustiveness checker will catch it).

#### Closed vs Open State Sets

Enums model **closed** sets of states — all possibilities are known at compile time. This is perfect for protocols, parsers, and business logic. For **open-ended** sets of states (e.g., plugin systems where new states can be added later), use trait objects instead:

```
Closed set (enum):  All variants known → exhaustive matching → maximum safety
Open set (trait):   New impls can be added → runtime dispatch → maximum flexibility
```

Real-world Rust libraries lean heavily on this pattern. The `serde` library models data formats with enums (`Value::String`, `Value::Number`, `Value::Array`, ...). The `http` crate models request methods (`Method::GET`, `Method::POST`, ...). The `syn` crate (Rust's own parser) represents the entire Rust syntax tree as nested enums.

The connection to formal methods is real: enum-based state machines bring the rigor of **formal verification** into everyday code — the compiler acts as a lightweight model checker, ensuring every state transition is accounted for.

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

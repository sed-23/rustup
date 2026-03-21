# Defining Structs 🏗️

> **Structs let you group related data together into a single, named type. They are the foundation of custom data types in Rust — think of them as blueprints for your data.**

---

## Table of Contents

- [Why Structs?](#why-structs)
- [Named-Field Structs](#named-field-structs)
  - [Defining a Struct](#defining-a-struct)
  - [Creating Instances](#creating-instances)
  - [Accessing Fields](#accessing-fields)
  - [Mutability](#mutability)
  - [Field Init Shorthand](#field-init-shorthand)
  - [Struct Update Syntax](#struct-update-syntax)
- [Tuple Structs](#tuple-structs)
  - [Newtype Pattern](#newtype-pattern)
- [Unit Structs](#unit-structs)
- [Struct Memory Layout](#struct-memory-layout)
  - [Stack vs Heap](#stack-vs-heap)
  - [Field Ordering and Padding](#field-ordering-and-padding)
- [Ownership and Structs](#ownership-and-structs)
  - [Structs That Own Data](#structs-that-own-data)
  - [Moving Structs](#moving-structs)
  - [Cloning Structs](#cloning-structs)
- [Printing Structs with Debug](#printing-structs-with-debug)
- [Common Patterns](#common-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Why Structs?

Without structs, you'd pass data around as separate variables — messy and error-prone:

```rust
// ❌ Unorganized — which string is which?
fn print_user(name: &str, email: &str, age: u32, active: bool) {
    println!("{} ({}) — age {}, active: {}", name, email, age, active);
}
```

With structs, related data is grouped together:

```rust
struct User {
    name: String,
    email: String,
    age: u32,
    active: bool,
}

// ✅ Clear, organized, self-documenting
fn print_user(user: &User) {
    println!("{} ({}) — age {}, active: {}", user.name, user.email, user.age, user.active);
}
```

---

## Named-Field Structs

### Defining a Struct

```rust
struct User {
    name: String,
    email: String,
    age: u32,
    active: bool,
}
```

Rules:
- Each field has a name and a type
- Fields are separated by commas
- Trailing comma is allowed (and encouraged by convention)
- Struct names use `PascalCase`, field names use `snake_case`

### Creating Instances

```rust
fn main() {
    let user = User {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        age: 30,
        active: true,
    };
}
```

**You must set every field.** There's no default:

```rust
// ❌ Missing fields
// let user = User {
//     name: String::from("Alice"),
// };  // ERROR: missing email, age, active
```

### Accessing Fields

Use dot notation:

```rust
fn main() {
    let user = User {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        age: 30,
        active: true,
    };
    
    println!("Name: {}", user.name);
    println!("Email: {}", user.email);
    println!("Age: {}", user.age);
    println!("Active: {}", user.active);
}
```

### Mutability

The entire struct instance is mutable or not — you can't mark individual fields as mutable:

```rust
fn main() {
    let mut user = User {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        age: 30,
        active: true,
    };
    
    user.age = 31;  // ✅ Instance is mut
    user.email = String::from("alice@newmail.com");
    
    println!("{}, age {}", user.name, user.age);
}
```

### Field Init Shorthand

When a variable has the same name as a struct field, you can use shorthand:

```rust
fn create_user(name: String, email: String) -> User {
    User {
        name,     // Same as name: name
        email,    // Same as email: email
        age: 0,
        active: true,
    }
}
```

### Struct Update Syntax

Create a new struct based on an existing one, changing only some fields:

```rust
fn main() {
    let user1 = User {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        age: 30,
        active: true,
    };
    
    // Create user2, changing only email
    let user2 = User {
        email: String::from("bob@example.com"),
        ..user1  // Fill remaining fields from user1
    };
    
    // ⚠️ user1.name was MOVED into user2!
    // println!("{}", user1.name);   // ❌ Error: value moved
    println!("{}", user1.age);       // ✅ u32 is Copy
    println!("{}", user1.active);    // ✅ bool is Copy
    println!("{}", user2.name);      // "Alice"
    println!("{}", user2.email);     // "bob@example.com"
}
```

**Important:** Struct update syntax uses **move** for non-Copy fields. Only Copy types (like `u32`, `bool`) remain usable on the original.

---

## Tuple Structs

Structs with unnamed fields — accessed by index instead of name:

```rust
struct Color(u8, u8, u8);
struct Point(f64, f64, f64);

fn main() {
    let red = Color(255, 0, 0);
    let origin = Point(0.0, 0.0, 0.0);
    
    println!("R: {}, G: {}, B: {}", red.0, red.1, red.2);
    println!("x: {}, y: {}, z: {}", origin.0, origin.1, origin.2);
}
```

Tuple structs create **distinct types**:

```rust
struct Meters(f64);
struct Kilometers(f64);

fn main() {
    let m = Meters(100.0);
    let km = Kilometers(5.0);
    
    // let x: Meters = km;  // ❌ Type mismatch! Even though both wrap f64
}
```

### Newtype Pattern

Wrapping a single value in a tuple struct to create a new type. This gives type safety at zero runtime cost:

```rust
struct UserId(u64);
struct OrderId(u64);

fn process_order(order_id: OrderId, user_id: UserId) {
    println!("Order {} for user {}", order_id.0, user_id.0);
}

fn main() {
    let user = UserId(42);
    let order = OrderId(1001);
    
    process_order(order, user);
    // process_order(user, order);  // ❌ Can't swap by accident!
}
```

---

## Unit Structs

Structs with no fields at all:

```rust
struct Marker;
struct AlwaysEqual;

fn main() {
    let m = Marker;
    let eq = AlwaysEqual;
}
```

Unit structs have zero size. They're useful when:
- Implementing a trait without storing data
- Using them as marker types
- Acting as sentinel values in type-level programming

---

## Struct Memory Layout

### Stack vs Heap

```rust
struct Point {
    x: f64,  // 8 bytes
    y: f64,  // 8 bytes
}

struct User {
    name: String,    // 24 bytes (ptr + len + cap)
    age: u32,        // 4 bytes
    active: bool,    // 1 byte
}
```

```
Point (entirely on stack):
┌─────────────┐
│ x: f64 (8B) │
│ y: f64 (8B) │
└─────────────┘
Total: 16 bytes on stack, 0 bytes on heap

User (stack metadata + heap data):
Stack:                              Heap:
┌─────────────────────┐            ┌───────────────┐
│ name.ptr ───────────────────────→│ A l i c e     │
│ name.len: 5         │            └───────────────┘
│ name.cap: 5         │
│ age: 30             │
│ active: true        │
└─────────────────────┘
```

### Field Ordering and Padding

Rust may **reorder fields** in memory for better alignment (unless you use `#[repr(C)]`):

```rust
// Logical order:
struct Example {
    a: u8,    // 1 byte
    b: u64,   // 8 bytes
    c: u8,    // 1 byte
}

// Without reordering (C layout): 24 bytes due to padding
// ┌────┬───────┬────────┬────┬───────┐
// │ a  │ pad7  │   b    │ c  │ pad7  │
// └────┴───────┴────────┴────┴───────┘
// 1 + 7 + 8 + 1 + 7 = 24 bytes

// With Rust's reordering: 16 bytes
// ┌────────┬────┬────┬──────┐
// │   b    │ a  │ c  │ pad6 │
// └────────┴────┴────┴──────┘
// 8 + 1 + 1 + 6 = 16 bytes
```

You can check sizes:

```rust
use std::mem;

fn main() {
    println!("Point: {} bytes", mem::size_of::<Point>());   // 16
    println!("User: {} bytes", mem::size_of::<User>());     // 56 (approx)
    println!("bool: {} bytes", mem::size_of::<bool>());     // 1
}
```

---

## Ownership and Structs

### Structs That Own Data

When a struct contains `String`, `Vec`, or other owned types, it **owns** that data:

```rust
struct Article {
    title: String,
    body: String,
    tags: Vec<String>,
}

fn main() {
    let article = Article {
        title: String::from("Hello Rust"),
        body: String::from("Rust is great!"),
        tags: vec![String::from("rust"), String::from("programming")],
    };
    
    // article owns title, body, and tags
    // When article goes out of scope, all are dropped
}
```

### Moving Structs

Structs with non-Copy fields are **moved**, not copied:

```rust
fn main() {
    let user1 = User {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        age: 30,
        active: true,
    };
    
    let user2 = user1;  // MOVE! user1 is now invalid
    // println!("{}", user1.name);  // ❌ Error
    println!("{}", user2.name);     // ✅ "Alice"
}
```

### Cloning Structs

To deep copy, derive `Clone`:

```rust
#[derive(Clone)]
struct User {
    name: String,
    email: String,
    age: u32,
    active: bool,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        age: 30,
        active: true,
    };
    
    let user2 = user1.clone();  // Deep copy
    println!("{} {}", user1.name, user2.name);  // Both work!
}
```

---

## Printing Structs with Debug

By default, structs can't be printed. Derive `Debug` to enable `{:?}`:

```rust
#[derive(Debug)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    
    println!("{:?}", p);    // Point { x: 3.0, y: 4.0 }
    println!("{:#?}", p);   // Pretty-printed:
    // Point {
    //     x: 3.0,
    //     y: 4.0,
    // }
    
    // dbg! macro — prints to stderr with file and line info
    dbg!(&p);
    // [src/main.rs:15] &p = Point {
    //     x: 3.0,
    //     y: 4.0,
    // }
}
```

---

## Common Patterns

### Builder Pattern (Simple Version)

```rust
struct Config {
    host: String,
    port: u16,
    max_connections: u32,
    timeout_secs: u64,
}

impl Config {
    fn new(host: &str) -> Config {
        Config {
            host: host.to_string(),
            port: 8080,
            max_connections: 100,
            timeout_secs: 30,
        }
    }
}

fn main() {
    let config = Config::new("localhost");
    println!("{}:{}", config.host, config.port);
}
```

### Nested Structs

```rust
#[derive(Debug)]
struct Address {
    street: String,
    city: String,
    country: String,
}

#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
    address: Address,
}

fn main() {
    let person = Person {
        name: String::from("Alice"),
        age: 30,
        address: Address {
            street: String::from("123 Main St"),
            city: String::from("Rustville"),
            country: String::from("Ferris Land"),
        },
    };
    
    println!("{} lives in {}", person.name, person.address.city);
}
```

---

## Exercises

### Exercise 1: Define and Use a Struct

Create a `Rectangle` struct with `width` and `height` fields (both `f64`). Write a function `area` that takes a `&Rectangle` and returns its area.

```rust
fn main() {
    let rect = Rectangle { width: 10.0, height: 5.0 };
    println!("Area: {}", area(&rect));  // 50.0
}
```

<details>
<summary>Solution</summary>

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

fn area(rect: &Rectangle) -> f64 {
    rect.width * rect.height
}
```

</details>

### Exercise 2: Struct Update Syntax

Given this code, predict what prints and why:

```rust
#[derive(Debug)]
struct Config {
    debug: bool,
    level: u32,
    name: String,
}

fn main() {
    let c1 = Config {
        debug: true,
        level: 3,
        name: String::from("app"),
    };
    
    let c2 = Config {
        level: 5,
        ..c1
    };
    
    println!("{}", c1.debug);   // ?
    println!("{}", c1.level);   // ?
    // println!("{}", c1.name); // ?
    println!("{:?}", c2);       // ?
}
```

<details>
<summary>Answer</summary>

- `c1.debug` → `true` (bool is Copy)
- `c1.level` → `3` (u32 is Copy)
- `c1.name` → ❌ ERROR — `name` was moved into `c2`
- `c2` → `Config { debug: true, level: 5, name: "app" }`

</details>

---

## Summary

| Struct Kind | Syntax | Access | Use Case |
|-------------|--------|--------|----------|
| Named fields | `struct S { x: T }` | `s.x` | Most data types |
| Tuple struct | `struct S(T, U)` | `s.0, s.1` | Newtype pattern, quick grouping |
| Unit struct | `struct S;` | N/A | Marker types, trait impl |

**Key Takeaways:**
1. Structs group related data under a single type
2. All fields must be initialized — no defaults
3. `mut` applies to the entire instance, not individual fields
4. Structs with non-Copy fields are moved, not copied
5. Derive `Debug` to print, `Clone` to deep copy
6. Struct update syntax (`..other`) moves non-Copy fields

---

**Next:** [Methods & Associated Functions →](./02-methods-and-impl.md)

<p align="center"><i>Tutorial 1 of 8 — Stage 4: Structuring Data</i></p>

# Methods & Associated Functions 🔧

> **Methods give structs behavior. With `impl` blocks, your data types stop being passive containers and become active participants in your program.**

---

## Table of Contents

- [What Is an impl Block?](#what-is-an-impl-block)
- [Defining Methods](#defining-methods)
  - [The &self Parameter](#the-self-parameter)
  - [Methods with &mut self](#methods-with-mut-self)
  - [Methods That Take Ownership (self)](#methods-that-take-ownership-self)
- [Methods with Additional Parameters](#methods-with-additional-parameters)
- [Associated Functions (Constructors)](#associated-functions-constructors)
- [Multiple impl Blocks](#multiple-impl-blocks)
- [Method Call Syntax and Auto-Referencing](#method-call-syntax-and-auto-referencing)
- [Methods and Ownership Recap](#methods-and-ownership-recap)
- [Getters and Setters](#getters-and-setters)
- [Builder Pattern](#builder-pattern)
- [Method Chaining](#method-chaining)
- [Common Patterns](#common-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is an `impl` Block?

An **implementation block** attaches functions to a struct:

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    // Everything inside here belongs to Rectangle
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

fn main() {
    let rect = Rectangle { width: 10.0, height: 5.0 };
    println!("Area: {}", rect.area());  // 50.0
}
```

---

## Defining Methods

### The `&self` Parameter

Methods receive the instance they're called on as their first parameter. `&self` means "borrow the instance immutably":

```rust
impl Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    
    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }
    
    fn is_square(&self) -> bool {
        (self.width - self.height).abs() < f64::EPSILON
    }
}
```

`&self` is shorthand for `self: &Self`, and `Self` is the type being implemented (`Rectangle`).

### Methods with `&mut self`

Methods that need to modify the struct take `&mut self`:

```rust
impl Rectangle {
    fn double_size(&mut self) {
        self.width *= 2.0;
        self.height *= 2.0;
    }
    
    fn set_width(&mut self, width: f64) {
        self.width = width;
    }
}

fn main() {
    let mut rect = Rectangle { width: 10.0, height: 5.0 };
    println!("Before: {} x {}", rect.width, rect.height);
    
    rect.double_size();
    println!("After: {} x {}", rect.width, rect.height);  // 20 x 10
}
```

### Methods That Take Ownership (`self`)

Sometimes a method consumes the instance:

```rust
struct Message {
    content: String,
}

impl Message {
    // Takes ownership — self is moved
    fn send(self) {
        println!("Sending: {}", self.content);
        // self is dropped at end of this function
    }
}

fn main() {
    let msg = Message { content: String::from("Hello!") };
    msg.send();
    // msg.send();  // ❌ msg was moved in the first call
}
```

Use `self` when the method:
- Transforms the type into something else
- Consumes the resource (sending a message, closing a connection)
- Returns `Self` for builder pattern

---

## Methods with Additional Parameters

```rust
impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
    
    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }
}

fn main() {
    let big = Rectangle { width: 20.0, height: 15.0 };
    let small = Rectangle { width: 10.0, height: 5.0 };
    
    println!("big holds small: {}", big.can_hold(&small));  // true
    println!("small holds big: {}", small.can_hold(&big));  // false
}
```

---

## Associated Functions (Constructors)

Functions inside `impl` that **don't take `self`** are called **associated functions**. They're called with `::` syntax:

```rust
impl Rectangle {
    // Associated function — no self parameter
    fn new(width: f64, height: f64) -> Rectangle {
        Rectangle { width, height }
    }
    
    fn square(size: f64) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    let rect = Rectangle::new(10.0, 5.0);      // :: syntax
    let square = Rectangle::square(8.0);
    
    println!("rect area: {}", rect.area());
    println!("square area: {}", square.area());
}
```

The most common associated function is `new` — Rust's convention for constructors.

---

## Multiple `impl` Blocks

A struct can have multiple `impl` blocks:

```rust
struct Circle {
    radius: f64,
}

impl Circle {
    fn new(radius: f64) -> Circle {
        Circle { radius }
    }
}

impl Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    
    fn circumference(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }
}
```

This is useful when implementing traits (covered later), but for now, one `impl` block is fine.

---

## Method Call Syntax and Auto-Referencing

Rust has **automatic referencing and dereferencing**. When you call `object.method()`, Rust automatically adds `&`, `&mut`, or `*` to make it match:

```rust
impl Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

fn main() {
    let rect = Rectangle::new(10.0, 5.0);
    
    // These are all the same:
    let a1 = rect.area();       // Rust auto-adds & → (&rect).area()
    let a2 = (&rect).area();    // Explicit borrow
    let a3 = Rectangle::area(&rect);  // Fully qualified
}
```

This is why you can call `&self` methods on an owned value and `&mut self` methods on a mutable-owned value — Rust figures it out.

---

## Methods and Ownership Recap

```rust
impl Rectangle {
    fn describe(&self) -> String {
        // &self: borrows immutably. Caller keeps ownership.
        format!("{}x{}", self.width, self.height)
    }
    
    fn grow(&mut self, amount: f64) {
        // &mut self: borrows mutably. Caller keeps ownership but lends write access.
        self.width += amount;
        self.height += amount;
    }
    
    fn into_square(self) -> Rectangle {
        // self: takes ownership. Caller loses the value.
        let side = self.width.max(self.height);
        Rectangle { width: side, height: side }
    }
}
```

| `self` form | Borrows? | Can modify? | Caller keeps value? |
|-------------|----------|-------------|---------------------|
| `&self` | Yes (immutable) | No | Yes |
| `&mut self` | Yes (mutable) | Yes | Yes |
| `self` | No (takes ownership) | Yes | No (moved) |

---

## Getters and Setters

Rust doesn't have private fields by default (within the same module), but in modules you'd use getters:

```rust
pub struct Temperature {
    celsius: f64,  // Private by default in a module
}

impl Temperature {
    pub fn new(celsius: f64) -> Temperature {
        Temperature { celsius }
    }
    
    // Getter
    pub fn celsius(&self) -> f64 {
        self.celsius
    }
    
    // Computed getter
    pub fn fahrenheit(&self) -> f64 {
        self.celsius * 9.0 / 5.0 + 32.0
    }
    
    // Setter with validation
    pub fn set_celsius(&mut self, value: f64) {
        if value < -273.15 {
            panic!("Temperature below absolute zero!");
        }
        self.celsius = value;
    }
}
```

---

## Builder Pattern

For structs with many fields, some optional:

```rust
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    timeout: u64,
}

struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: u32,
    timeout: u64,
}

impl ServerBuilder {
    fn new(host: &str) -> ServerBuilder {
        ServerBuilder {
            host: host.to_string(),
            port: 8080,
            max_connections: 100,
            timeout: 30,
        }
    }
    
    fn port(mut self, port: u16) -> ServerBuilder {
        self.port = port;
        self
    }
    
    fn max_connections(mut self, max: u32) -> ServerBuilder {
        self.max_connections = max;
        self
    }
    
    fn timeout(mut self, secs: u64) -> ServerBuilder {
        self.timeout = secs;
        self
    }
    
    fn build(self) -> Server {
        Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
            timeout: self.timeout,
        }
    }
}

fn main() {
    let server = ServerBuilder::new("0.0.0.0")
        .port(3000)
        .max_connections(500)
        .timeout(60)
        .build();
    
    println!("Server on {}:{}", server.host, server.port);
}
```

---

## Method Chaining

Methods that return `&mut self` or `Self` enable chaining:

```rust
struct QueryBuilder {
    table: String,
    conditions: Vec<String>,
    limit: Option<u32>,
}

impl QueryBuilder {
    fn from(table: &str) -> QueryBuilder {
        QueryBuilder {
            table: table.to_string(),
            conditions: Vec::new(),
            limit: None,
        }
    }
    
    fn where_clause(&mut self, condition: &str) -> &mut Self {
        self.conditions.push(condition.to_string());
        self
    }
    
    fn limit(&mut self, n: u32) -> &mut Self {
        self.limit = Some(n);
        self
    }
    
    fn to_sql(&self) -> String {
        let mut sql = format!("SELECT * FROM {}", self.table);
        if !self.conditions.is_empty() {
            sql.push_str(" WHERE ");
            sql.push_str(&self.conditions.join(" AND "));
        }
        if let Some(limit) = self.limit {
            sql.push_str(&format!(" LIMIT {}", limit));
        }
        sql
    }
}

fn main() {
    let query = QueryBuilder::from("users")
        .where_clause("age > 18")
        .where_clause("active = true")
        .limit(10)
        .to_sql();
    
    println!("{}", query);
    // SELECT * FROM users WHERE age > 18 AND active = true LIMIT 10
}
```

---

## Common Patterns

### Display Trait (Custom Printing)

```rust
use std::fmt;

struct Point {
    x: f64,
    y: f64,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("{}", p);  // (3.0, 4.0)
}
```

### Converting Between Types

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl Celsius {
    fn to_fahrenheit(&self) -> Fahrenheit {
        Fahrenheit(self.0 * 9.0 / 5.0 + 32.0)
    }
}

impl Fahrenheit {
    fn to_celsius(&self) -> Celsius {
        Celsius((self.0 - 32.0) * 5.0 / 9.0)
    }
}
```

---

## Exercises

### Exercise 1: Implement Methods

Add methods to this struct:

```rust
struct Counter {
    count: u32,
}

// Implement:
// - Counter::new() → Counter (starts at 0)
// - counter.increment() → adds 1
// - counter.decrement() → subtracts 1 (minimum 0)
// - counter.value() → returns current count
// - counter.reset() → sets count to 0

fn main() {
    let mut c = Counter::new();
    c.increment();
    c.increment();
    c.increment();
    c.decrement();
    assert_eq!(c.value(), 2);
    c.reset();
    assert_eq!(c.value(), 0);
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
    
    fn increment(&mut self) {
        self.count += 1;
    }
    
    fn decrement(&mut self) {
        if self.count > 0 {
            self.count -= 1;
        }
    }
    
    fn value(&self) -> u32 {
        self.count
    }
    
    fn reset(&mut self) {
        self.count = 0;
    }
}
```

</details>

### Exercise 2: Which Self?

For each method, decide if it should take `&self`, `&mut self`, or `self`:

- `fn name(&self) -> &str` — returns the name
- `fn set_name(??? self, new_name: String)` — changes the name
- `fn into_parts(??? self) -> (String, u32)` — destructures the struct
- `fn is_valid(&self) -> bool` — checks validity

<details>
<summary>Answer</summary>

- `name` → `&self` (read-only)
- `set_name` → `&mut self` (modifies)
- `into_parts` → `self` (consumes, returning owned parts)
- `is_valid` → `&self` (read-only)

</details>

---

## Summary

| Concept | Syntax | Purpose |
|---------|--------|---------|
| Method | `fn method(&self)` | Access data on an instance |
| Mutable method | `fn method(&mut self)` | Modify instance data |
| Consuming method | `fn method(self)` | Take ownership of instance |
| Associated function | `fn func() -> Self` | Constructors, no instance needed |
| Call method | `instance.method()` | Dot notation |
| Call associated fn | `Type::func()` | Double-colon notation |

**Key Takeaways:**
1. `impl` blocks attach behavior to structs
2. `&self` for reading, `&mut self` for writing, `self` for consuming
3. Associated functions (no `self`) are like static methods — use `::` to call
4. Rust auto-references when calling methods
5. `new()` is the convention for constructors
6. Method chaining and builder pattern are idiomatic Rust

---

**Next:** [Enums →](./03-enums.md)

<p align="center"><i>Tutorial 2 of 8 — Stage 4: Structuring Data</i></p>

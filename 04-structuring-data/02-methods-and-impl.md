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

### Methods Across Programming Languages

Rust's `impl` block approach to methods is just one of many designs languages have chosen. Understanding how other languages attach behavior to data reveals *why* Rust made its particular tradeoffs.

**C — No methods at all.** You write standalone functions and pass a pointer to the struct manually:

```c
typedef struct { double width; double height; } Rectangle;

double rect_area(const Rectangle* r) {
    return r->width * r->height;
}

rect_area(&my_rect);  // caller must pass the pointer
```

This is essentially what Rust's `&self` desugars to — but Rust gives you dot-call syntax on top.

**C++ — Methods live inside the class.** A hidden `this` pointer is passed implicitly:

```cpp
class Rectangle {
public:
    double area() { return this->width * this->height; }
    //                     ^^^^ implicit, often omitted
};
```

**Java / C# — Everything is a method.** Standalone functions don't exist; even `main` lives inside a class. The receiver is the implicit `this`.

**Python — Explicit `self`, like Rust!** Guido van Rossum famously insisted that the receiver be an explicit parameter. This was controversial but makes the data flow visible:

```python
class Rectangle:
    def area(self):          # self is explicit
        return self.width * self.height
```

**Go — Receiver syntax outside the struct.** Go's approach is the closest cousin to Rust's `impl` blocks. Methods are defined *outside* the struct, with a "receiver" parameter:

```go
func (r Rectangle) area() float64 {
    return r.width * r.height
}
```

**JavaScript — `this` binding chaos.** Methods can be defined via prototypes or class syntax, but `this` depends on *how* the function is called, not *where* it's defined — a notorious source of bugs.

#### Comparison Table

| Language | Where Methods Live | Self Parameter | Standalone Fns? | Ownership in Signature? |
|------------|------------------------------|----------------------|-----------------|-------------------------|
| C | N/A (no methods) | Manual pointer | Yes (only fns) | No |
| C++ | Inside `class {}` | Implicit `this` | Yes | No |
| Java / C# | Inside `class {}` | Implicit `this` | No | No |
| Python | Inside `class:` | Explicit `self` | Yes | No |
| Go | Outside struct (receiver) | Explicit receiver | Yes | Pointer vs value recv |
| JavaScript | Inside `class {}` / prototype| Implicit `this` | Yes | No |
| **Rust** | **Separate `impl` block** | **Explicit `&self`** | **Yes** | **Yes (`&`, `&mut`, owned)** |

> **The `self` debate:** Python and Rust chose *explicit* self — you always see what's being borrowed.
> Java and C++ chose *implicit* this — less typing, but the data flow is hidden.
> Rust goes further: the `self` parameter encodes **ownership semantics** (`&self` vs `&mut self` vs `self`),
> something no other mainstream language does.

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

### How Method Dispatch Works Under the Hood

When you write `rect.area()`, what actually happens at the machine level? Understanding dispatch demystifies method calls and explains Rust's zero-cost abstraction promise.

#### Static Dispatch (What You Get by Default)

At compile time, the compiler knows the **exact concrete type** of `rect`. It resolves `rect.area()` to a specific function — no different from a plain function call in C:

```
  Your Rust code          What the compiler generates
  ─────────────           ──────────────────────────
  rect.area()      →      Rectangle_area(&rect)
                           ^^^^^^^^^^^^^^^^^^^^
                           Direct call, known at compile time
```

This is called **static dispatch** (or monomorphization when generics are involved). The compiler can **inline** the function body directly at the call site, enabling further optimizations like constant folding and loop unrolling. There is **zero runtime overhead** — it's as fast as hand-written C.

#### Auto-Referencing: The Method Resolution Algorithm

When you write `rect.area()` and `area` expects `&self`, Rust must figure out how to convert `rect` (an owned `Rectangle`) into `&Rectangle`. It uses this search order:

```
  Rust tries these transformations in order:
  ┌─────────────────────────────────────────┐
  │ 1.  T         →  Does T have area()?    │
  │ 2.  &T        →  Does &T have area()?   │  ← matches &self
  │ 3.  &mut T    →  Does &mut T have area()?│
  │ 4.  deref(T)  →  repeat from step 1     │
  │ 5.  &deref(T) →  repeat from step 2     │
  └─────────────────────────────────────────┘
```

This is why you *never* need to write `(&rect).area()` — the compiler inserts the `&` for you. It also explains why calling methods through `Box<T>`, `Rc<T>`, or `Arc<T>` "just works": auto-deref unwraps the smart pointer, then auto-ref matches the method signature.

#### Dynamic Dispatch (A Preview)

Later, when you learn about trait objects (`&dyn Trait`), you'll encounter **dynamic dispatch**. Here the exact function isn't known at compile time — instead, the compiler generates a **vtable** (virtual method table):

```
  Static dispatch (default)        Dynamic dispatch (opt-in)
  ─────────────────────────        ────────────────────────
  rect.area()                      shape.area()  // shape: &dyn Shape
       │                                │
       ▼                                ▼
  Rectangle_area(&rect)            vtable_ptr → ┌──────────────┐
  (direct call, inlineable)                      │ area: 0x4A20 │─→ Rectangle_area
                                                 │ perimeter: …  │
                                                 └──────────────┘
                                                 (pointer indirection, ~1-2ns cost)
```

| Property | Static Dispatch | Dynamic Dispatch |
|-------------------|---------------------------|---------------------------|
| Resolved at | Compile time | Runtime |
| Function call | Direct (inlineable) | Indirect via vtable ptr |
| Runtime cost | Zero | ~1-2ns per call |
| Binary size | Larger (monomorphized) | Smaller (shared code) |
| Opt-in syntax | Default everywhere | `dyn Trait` |

> **Key insight:** Rust defaults to static dispatch everywhere. You must *explicitly* opt into dynamic dispatch with `dyn`. This is the opposite of Java/C++, where virtual/dynamic dispatch is the default and you opt *out* with `final`/non-virtual.

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

### The Builder Pattern in Depth

The builder above works, but *why* does this pattern exist, and when should you actually use it in Rust?

#### The Core Problem: Too Many Fields

Imagine a struct with 12 fields, 8 of which have sensible defaults. A `new()` function with 12 parameters is unusable:

```rust
// ❌ What are these arguments? Impossible to read.
let server = Server::new("0.0.0.0", 3000, 500, 60, true, false, None, None, 4, "info", true, "/tmp");
```

Different languages solve this differently:

| Language | Solution for Many Optional Fields |
|------------|------------------------------------------------------|
| Python | Keyword arguments: `Server(host="0.0.0.0", port=3000)` |
| Kotlin | Data classes with default parameter values |
| Java / C# | Builder pattern (the *only* clean option) |
| Go | Functional options: `NewServer(WithPort(3000))` |
| **Rust** | **Struct literals + Default, OR builder pattern** |

#### Rust's Built-In Alternative: `Default` + Struct Update Syntax

For many cases, you don't need a builder at all. The `Default` trait + struct update syntax gives you named fields with defaults:

```rust
#[derive(Default)]
struct ServerConfig {
    host: String,         // defaults to ""
    port: u16,            // defaults to 0
    max_connections: u32, // defaults to 0
    timeout: u64,         // defaults to 0
}

let config = ServerConfig {
    host: "0.0.0.0".to_string(),
    port: 3000,
    ..Default::default()  // fill the rest with defaults
};
```

This is simpler than a builder and perfectly idiomatic. Use it when defaults are trivial.

#### When Builders ARE Worth It

Builders shine when you need something `Default` can't give you:

1. **Validation during construction** — a `build()` method can return `Result<T, Error>`, rejecting invalid configurations
2. **Complex defaults** — e.g., port defaults to 8080 (not 0), host defaults to `"localhost"`
3. **Fluent API design** — chained calls read like a configuration DSL
4. **Required vs optional fields** — the builder can enforce that certain fields *must* be set

#### The Typestate Builder Pattern (Advanced)

You can use Rust's type system to enforce at **compile time** that required fields are set before `build()` can be called:

```rust
// Marker types
struct NoHost;
struct HasHost(String);

struct ServerBuilder<H> {
    host: H,          // generic — either NoHost or HasHost
    port: u16,
}

impl ServerBuilder<NoHost> {
    fn new() -> Self {
        ServerBuilder { host: NoHost, port: 8080 }
    }
    fn host(self, h: &str) -> ServerBuilder<HasHost> {
        ServerBuilder { host: HasHost(h.to_string()), port: self.port }
    }
}

impl ServerBuilder<HasHost> {
    fn build(self) -> Server { /* only available when host is set! */ }
}
```

This way, calling `.build()` without first calling `.host()` is a **compile error**, not a runtime panic.

#### Real-World Builders in the Rust Ecosystem

- **`reqwest::ClientBuilder`** — configure HTTP timeouts, proxies, TLS settings, then `.build()?`
- **`tokio::runtime::Builder`** — choose single-threaded vs multi-threaded, set worker count, then `.build()?`
- **`clap::Command`** (CLI parser) — chain `.arg()`, `.subcommand()`, then `.get_matches()`

All of these return `Result` from `.build()` — combining the builder pattern with Rust's error handling.

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

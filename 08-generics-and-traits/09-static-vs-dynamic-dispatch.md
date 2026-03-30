# Static vs Dynamic Dispatch ⚡

> **"Dispatch is the art of choosing — at compile time or runtime — which function body to run for a given call."**

---

## Table of Contents

1. [What Is Dispatch?](#1-what-is-dispatch)
2. [Static Dispatch (Monomorphization)](#2-static-dispatch-monomorphization)
3. [Dynamic Dispatch (Trait Objects)](#3-dynamic-dispatch-trait-objects)
4. [The Vtable — Under the Hood](#4-the-vtable--under-the-hood)
5. [Trade-Offs at a Glance](#5-trade-offs-at-a-glance)
6. [When to Use Static Dispatch](#6-when-to-use-static-dispatch)
7. [When to Use Dynamic Dispatch](#7-when-to-use-dynamic-dispatch)
8. [Object Safety](#8-object-safety)
9. [Dispatch in Other Languages](#9-dispatch-in-other-languages)
10. [Real-World Examples](#10-real-world-examples)
11. [The `impl Trait` Return Type](#11-the-impl-trait-return-type)
12. [Exercises](#12-exercises)
13. [Summary](#13-summary)

---

## 1. What Is Dispatch?

When you call a method on a trait, Rust needs to figure out *which concrete function*
to execute. A `Shape` trait might have `area()`, but `Circle::area` and
`Rectangle::area` are completely different functions living at different addresses
in your binary.

**Dispatch** is the mechanism that resolves that question:

```
trait Shape { fn area(&self) -> f64; }

struct Circle { radius: f64 }
struct Rectangle { width: f64, height: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 { self.width * self.height }
}
```

Given some variable `s` that satisfies `Shape`, calling `s.area()` must jump to
the right implementation. Rust offers two strategies:

| Strategy           | Resolution Time | Keyword / Syntax        |
|--------------------|-----------------|-------------------------|
| **Static dispatch**  | Compile time    | `impl Trait`, `T: Trait` |
| **Dynamic dispatch** | Runtime         | `dyn Trait`              |

Let's explore each one in depth.

---

## 2. Static Dispatch (Monomorphization)

### The Idea

With static dispatch the compiler knows the concrete type at every call site.
It **stamps out a separate copy** of the function for each type — a process called
**monomorphization** (literally "making one form").

### Syntax — Two Equivalent Forms

```rust
// Form 1: impl Trait (syntactic sugar)
fn print_area(shape: &impl Shape) {
    println!("Area: {:.2}", shape.area());
}

// Form 2: Explicit generic with trait bound
fn print_area<T: Shape>(shape: &T) {
    println!("Area: {:.2}", shape.area());
}
```

Both forms are identical after compilation. The compiler sees every call site:

```rust
let c = Circle { radius: 5.0 };
let r = Rectangle { width: 3.0, height: 4.0 };

print_area(&c);  // compiler generates print_area_Circle
print_area(&r);  // compiler generates print_area_Rectangle
```

### What the Compiler Actually Generates

Conceptually, monomorphization turns the generic function into specialized versions:

```rust
// Generated (pseudo-code) — you never write this yourself
fn print_area_Circle(shape: &Circle) {
    println!("Area: {:.2}", Circle::area(shape));
}

fn print_area_Rectangle(shape: &Rectangle) {
    println!("Area: {:.2}", Rectangle::area(shape));
}
```

Each version contains a **direct function call** — no indirection, no pointer
chasing. The CPU can even inline the call entirely.

### Characteristics

- **Zero runtime overhead** — every call is resolved at compile time.
- **Enables inlining** — the optimizer can inline trait method bodies.
- **Increases binary size** — N types → N copies of the function.
- **Increases compile time** — the compiler does more work up front.

### A Larger Example

```rust
use std::fmt::Display;

fn log_twice<T: Display>(value: &T) {
    println!("[LOG] {}", value);
    println!("[LOG] {}", value);
}

fn main() {
    log_twice(&42_i32);       // generates log_twice<i32>
    log_twice(&"hello");      // generates log_twice<&str>
    log_twice(&3.14_f64);     // generates log_twice<f64>
}
```

Three separate machine-code functions are emitted — each optimized for its
concrete type.

---

## 3. Dynamic Dispatch (Trait Objects)

### The Idea

Sometimes you **don't know** (or don't care about) the concrete type. You just
want "anything that implements `Shape`." Dynamic dispatch lets you call trait
methods through a **pointer** that is resolved at **runtime**.

### Syntax — `dyn Trait`

```rust
fn print_area(shape: &dyn Shape) {
    println!("Area: {:.2}", shape.area());
}
```

There is only **one** compiled copy of `print_area`. When you call `shape.area()`,
the program looks up the correct function pointer in a **vtable** at runtime.

### Calling It

```rust
let c = Circle { radius: 5.0 };
let r = Rectangle { width: 3.0, height: 4.0 };

print_area(&c as &dyn Shape);
print_area(&r as &dyn Shape);

// Rust can also coerce automatically:
print_area(&c);
print_area(&r);
```

### Owned Trait Objects

References aren't the only way. You can also use `Box`, `Rc`, or `Arc`:

```rust
let shapes: Vec<Box<dyn Shape>> = vec![
    Box::new(Circle { radius: 2.0 }),
    Box::new(Rectangle { width: 5.0, height: 3.0 }),
];

for shape in &shapes {
    println!("Area: {:.2}", shape.area());
}
```

This is **the** canonical use case: a **heterogeneous collection** of different
types behind a single trait interface.

### Characteristics

- **Small binary** — only one copy of each function that accepts `dyn Trait`.
- **Flexible** — new types implementing the trait can be added without recompiling.
- **Runtime cost** — every trait method call involves an indirect jump through
  the vtable (typically a pointer dereference + call).
- **No inlining** — the optimizer generally cannot inline through a vtable call.

---

## 4. The Vtable — Under the Hood

A `&dyn Shape` is **not** a regular thin pointer. It is a **fat pointer** — two
machine words wide:

```
        &dyn Shape (fat pointer)
        ┌─────────────────────────────────┐
        │  ptr_to_data: *const ()         │ ──► actual Circle / Rectangle / …
        │  ptr_to_vtable: *const Vtable   │ ──► vtable for that concrete type
        └─────────────────────────────────┘
```

### What Lives in the Vtable?

The vtable is a static, read-only table generated by the compiler for each
`(ConcreteType, Trait)` pair:

```
     Vtable for (Circle, Shape)
     ┌────────────────────────────────────────┐
     │  drop_in_place : fn(*mut Circle)       │   ← destructor
     │  size          : usize (e.g., 8)       │   ← size of Circle
     │  align         : usize (e.g., 8)       │   ← alignment of Circle
     │  area          : fn(&Circle) -> f64    │   ← Shape::area
     └────────────────────────────────────────┘

     Vtable for (Rectangle, Shape)
     ┌────────────────────────────────────────┐
     │  drop_in_place : fn(*mut Rectangle)    │
     │  size          : usize (e.g., 16)      │
     │  align         : usize (e.g., 8)       │
     │  area          : fn(&Rectangle) -> f64 │
     └────────────────────────────────────────┘
```

### How a Trait Method Call Works

When you write `shape.area()` on a `&dyn Shape`:

```
1.  Load ptr_to_vtable from the fat pointer.
2.  Index into the vtable to find the `area` function pointer.
3.  Call that function pointer, passing ptr_to_data as `&self`.
```

In pseudo-assembly:

```
    ; rdi = ptr_to_data, rsi = ptr_to_vtable
    mov  rax, [rsi + offset_of_area]   ; load fn pointer from vtable
    mov  rdi, rdi                       ; self = ptr_to_data
    call rax                            ; indirect call
```

The cost is small — usually one extra memory load — but it **prevents inlining**,
which is often the bigger performance concern.

### Visualizing a `Vec<Box<dyn Shape>>`

```
  Stack                     Heap
  ┌──────────┐
  │ Vec data │─────►┌──────────────────────┐
  └──────────┘      │ Box<dyn Shape> #0    │
                    │  ├─ ptr_to_data ─────────► Circle { radius: 2.0 }
                    │  └─ ptr_to_vtable ───────► vtable(Circle, Shape)
                    │ Box<dyn Shape> #1    │
                    │  ├─ ptr_to_data ─────────► Rect { w: 5.0, h: 3.0 }
                    │  └─ ptr_to_vtable ───────► vtable(Rect, Shape)
                    └──────────────────────┘
```

Each element in the `Vec` is two pointers wide, regardless of the concrete type.

---

## 5. Trade-Offs at a Glance

| Dimension              | Static (`impl` / generics)         | Dynamic (`dyn Trait`)                  |
|------------------------|------------------------------------|----------------------------------------|
| **Resolution**         | Compile time                       | Runtime                                |
| **Mechanism**          | Monomorphization (code duplication)| Vtable (indirect call)                 |
| **Runtime cost**       | Zero — direct / inlined calls      | ~1 pointer load + indirect call        |
| **Binary size**        | Larger (N copies per N types)      | Smaller (one copy)                     |
| **Compile time**       | Slower (more code to generate)     | Faster                                 |
| **Inlining**           | Yes                                | No (generally)                         |
| **Heterogeneous collections** | Not possible directly        | Yes — `Vec<Box<dyn Trait>>`            |
| **Flexibility**        | Types must be known at compile time| New types can be added at runtime      |
| **Object safety required?** | No                            | Yes                                    |

The rule of thumb:

- **Default to static dispatch.** It's zero-cost and idiomatic.
- **Reach for dynamic dispatch** when you need heterogeneous storage or when
  binary size / compile time is a concern.

---

## 6. When to Use Static Dispatch

### Hot Paths

In performance-critical code, you want every optimization the compiler can give
you. Static dispatch allows inlining, loop unrolling, and constant propagation
across trait boundaries:

```rust
fn sum_areas<T: Shape>(shapes: &[T]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}
```

If `T = Circle`, the compiler can inline `Circle::area` and potentially
auto-vectorize the loop.

### Generic Data Structures

Standard library types like `HashMap<K, V>` use static dispatch everywhere.
The hash function, equality comparison, and allocation are all monomorphized:

```rust
use std::collections::HashMap;

let mut scores: HashMap<String, i32> = HashMap::new();
scores.insert("Alice".into(), 100);
```

### When All Types Are Known at Compile Time

If you're writing an application (not a library) and you know every concrete type
that will ever appear, static dispatch is the right default:

```rust
enum ShapeKind {
    Circle(Circle),
    Rectangle(Rectangle),
}

impl ShapeKind {
    fn area(&self) -> f64 {
        match self {
            ShapeKind::Circle(c) => c.area(),
            ShapeKind::Rectangle(r) => r.area(),
        }
    }
}
```

This **enum dispatch** pattern avoids both code duplication *and* vtable overhead,
but requires a closed set of types.

---

## 7. When to Use Dynamic Dispatch

### Heterogeneous Collections

The most common use case — storing different types in one container:

```rust
fn make_scene() -> Vec<Box<dyn Shape>> {
    vec![
        Box::new(Circle { radius: 1.0 }),
        Box::new(Rectangle { width: 2.0, height: 3.0 }),
        Box::new(Circle { radius: 4.5 }),
    ]
}

fn total_area(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}
```

### Plugin Systems

When new types can be loaded at runtime (e.g., from a shared library):

```rust
trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn execute(&self, input: &str) -> String;
}

struct PluginRegistry {
    plugins: Vec<Box<dyn Plugin>>,
}

impl PluginRegistry {
    fn register(&mut self, plugin: Box<dyn Plugin>) {
        println!("Registered plugin: {}", plugin.name());
        self.plugins.push(plugin);
    }

    fn run_all(&self, input: &str) {
        for plugin in &self.plugins {
            let output = plugin.execute(input);
            println!("[{}] {}", plugin.name(), output);
        }
    }
}
```

### Reducing Binary Size

In embedded or WASM contexts where binary size matters, dynamic dispatch can be
preferable. One copy of the function is smaller than N monomorphized copies.

### Reducing Compile Times

In large projects, excessive monomorphization can slow down builds significantly.
Boxing trait objects behind `dyn` can cut compile times by reducing the amount
of generic code the compiler needs to instantiate.

---

## 8. Object Safety

Not every trait can be used as `dyn Trait`. A trait must be **object-safe** to
appear behind `dyn`. The compiler enforces these rules because the vtable
mechanism has constraints.

### Rules for Object Safety

A trait is **object-safe** if *all* of the following hold:

| Rule | Why |
|------|-----|
| The trait does **not** require `Self: Sized` | `dyn Trait` is unsized — it can't satisfy `Sized` |
| No method returns `Self` | The concrete type is erased; the compiler can't know the return size |
| No method has generic type parameters | Generic methods would require infinite vtable entries |
| No associated functions without `self` (i.e., no static methods) | There's no instance to dispatch on |

### Examples

#### Object-safe ✅

```rust
trait Draw {
    fn draw(&self);
    fn bounding_box(&self) -> (f64, f64, f64, f64);
}

// This works:
let canvas: Vec<Box<dyn Draw>> = vec![/* ... */];
```

#### Not object-safe ❌ — Returns `Self`

```rust
trait Clonable {
    fn clone_self(&self) -> Self;
    //                       ^^^^ returns Self — not object-safe
}

// ERROR: the trait `Clonable` cannot be made into an object
// let items: Vec<Box<dyn Clonable>> = vec![];
```

#### Not object-safe ❌ — Generic method

```rust
trait Serializer {
    fn serialize<W: std::io::Write>(&self, writer: &mut W);
    //          ^^^^^^^^^^^^^^^^^^^ generic — not object-safe
}
```

#### Not object-safe ❌ — `Self: Sized` bound

```rust
trait Foo: Sized {
    //     ^^^^^ requires Self: Sized — not object-safe
    fn bar(&self);
}
```

### Workaround: `where Self: Sized` on Individual Methods

You can exclude specific methods from the vtable while keeping the trait
object-safe overall:

```rust
trait Clonable {
    fn name(&self) -> &str;

    // This method is excluded from dyn dispatch:
    fn clone_self(&self) -> Self
    where
        Self: Sized;
}

// Now this works:
let items: Vec<Box<dyn Clonable>> = vec![/* ... */];
// But you can't call clone_self() through &dyn Clonable.
```

---

## 9. Dispatch in Other Languages

Understanding how Rust's dispatch model compares to other languages helps
solidify the concepts.

| Language    | Default Dispatch     | Static Dispatch Mechanism        | Dynamic Dispatch Mechanism         |
|-------------|----------------------|----------------------------------|------------------------------------|
| **Rust**    | Static (generics)    | Monomorphization                 | `dyn Trait` + vtable               |
| **C++**     | Static (templates)   | Template instantiation           | `virtual` methods + vtable         |
| **Java**    | Dynamic              | JIT devirtualization (runtime)   | `virtual` by default + vtable      |
| **Go**      | Dynamic (interfaces) | None (no generics until 1.18)    | Interface value = (type, ptr) pair |
| **Python**  | Dynamic              | None                             | Attribute lookup at runtime        |
| **Haskell** | Static (by default)  | Specialization / dict. passing   | Existential types (`SomeClass =>`) |
| **Swift**   | Static (generics)    | Specialization + witness tables  | Protocol existentials (`any P`)    |

### C++ Comparison

C++ is the closest analog to Rust:

```cpp
// Static dispatch — template (like Rust generics)
template<typename T>
void draw(const T& shape) {
    shape.draw();  // resolved at compile time
}

// Dynamic dispatch — virtual (like Rust dyn)
class Shape {
public:
    virtual void draw() const = 0;  // pure virtual
    virtual ~Shape() = default;
};

void draw(const Shape& shape) {
    shape.draw();  // resolved at runtime via vtable
}
```

Key difference: in C++, you must design your class hierarchy for dynamic dispatch
upfront (inheritance + `virtual`). In Rust, *any* struct can implement *any* trait
after the fact — dispatch is decoupled from the type definition.

### Java Comparison

```java
// Java: everything is virtual by default
interface Shape {
    double area();  // always dispatched dynamically
}

// The JIT compiler may "devirtualize" hot calls,
// but the programmer has no control over this.
```

In Java, you **cannot** choose static dispatch at the language level. The JVM's
JIT may optimize it away, but that's an implementation detail.

### Go Comparison

```go
// Go: interfaces are always dynamic
type Shape interface {
    Area() float64
}

func printArea(s Shape) {
    fmt.Printf("Area: %.2f\n", s.Area())
    // Always dispatched dynamically via interface value
}
```

Go interfaces are structurally typed (no explicit `impl`) and always use dynamic
dispatch. There is no generics-based static dispatch in Go's interface system
(though Go 1.18+ added generics with type parameters, which enable static dispatch
separately from interfaces).

---

## 10. Real-World Examples

### Example 1: Drawing App with Shapes

```rust
use std::f64::consts::PI;

trait Shape {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;
    fn describe(&self) -> String;
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

struct Triangle {
    a: f64,
    b: f64,
    c: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }
    fn perimeter(&self) -> f64 {
        2.0 * PI * self.radius
    }
    fn describe(&self) -> String {
        format!("Circle(r={})", self.radius)
    }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }
    fn describe(&self) -> String {
        format!("Rect({}×{})", self.width, self.height)
    }
}

impl Shape for Triangle {
    fn area(&self) -> f64 {
        let s = (self.a + self.b + self.c) / 2.0;
        (s * (s - self.a) * (s - self.b) * (s - self.c)).sqrt()
    }
    fn perimeter(&self) -> f64 {
        self.a + self.b + self.c
    }
    fn describe(&self) -> String {
        format!("Triangle({}, {}, {})", self.a, self.b, self.c)
    }
}

// ── Static dispatch: when we know the type ──
fn print_shape_info<T: Shape>(shape: &T) {
    println!(
        "{}: area={:.2}, perimeter={:.2}",
        shape.describe(),
        shape.area(),
        shape.perimeter()
    );
}

// ── Dynamic dispatch: heterogeneous canvas ──
struct Canvas {
    shapes: Vec<Box<dyn Shape>>,
}

impl Canvas {
    fn new() -> Self {
        Canvas { shapes: Vec::new() }
    }

    fn add(&mut self, shape: Box<dyn Shape>) {
        self.shapes.push(shape);
    }

    fn total_area(&self) -> f64 {
        self.shapes.iter().map(|s| s.area()).sum()
    }

    fn render(&self) {
        for (i, shape) in self.shapes.iter().enumerate() {
            println!("[{}] {} — area: {:.2}", i, shape.describe(), shape.area());
        }
    }
}

fn main() {
    // Static dispatch
    let c = Circle { radius: 5.0 };
    let r = Rectangle { width: 3.0, height: 4.0 };
    print_shape_info(&c);
    print_shape_info(&r);

    // Dynamic dispatch
    let mut canvas = Canvas::new();
    canvas.add(Box::new(Circle { radius: 2.0 }));
    canvas.add(Box::new(Rectangle { width: 10.0, height: 5.0 }));
    canvas.add(Box::new(Triangle { a: 3.0, b: 4.0, c: 5.0 }));
    canvas.render();
    println!("Total area: {:.2}", canvas.total_area());
}
```

### Example 2: Event System

```rust
trait EventHandler {
    fn event_type(&self) -> &str;
    fn handle(&self, payload: &str);
}

struct Logger;
struct Notifier { recipient: String }

impl EventHandler for Logger {
    fn event_type(&self) -> &str { "any" }
    fn handle(&self, payload: &str) {
        println!("[LOG] {}", payload);
    }
}

impl EventHandler for Notifier {
    fn event_type(&self) -> &str { "alert" }
    fn handle(&self, payload: &str) {
        println!("Notifying {}: {}", self.recipient, payload);
    }
}

struct EventBus {
    handlers: Vec<Box<dyn EventHandler>>,
}

impl EventBus {
    fn new() -> Self {
        EventBus { handlers: Vec::new() }
    }

    fn subscribe(&mut self, handler: Box<dyn EventHandler>) {
        self.handlers.push(handler);
    }

    fn emit(&self, event_type: &str, payload: &str) {
        for handler in &self.handlers {
            if handler.event_type() == "any" || handler.event_type() == event_type {
                handler.handle(payload);
            }
        }
    }
}

fn main() {
    let mut bus = EventBus::new();
    bus.subscribe(Box::new(Logger));
    bus.subscribe(Box::new(Notifier {
        recipient: "admin@example.com".into(),
    }));

    bus.emit("alert", "Server CPU at 95%");
    bus.emit("info", "Deployment complete");
}
```

### Example 3: Plugin Architecture

```rust
trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn version(&self) -> &str;
    fn process(&self, data: &mut Vec<u8>);
}

struct CompressionPlugin;
struct EncryptionPlugin { key: String }

impl Plugin for CompressionPlugin {
    fn name(&self) -> &str { "compress" }
    fn version(&self) -> &str { "1.0.0" }
    fn process(&self, data: &mut Vec<u8>) {
        println!("  [compress] Processing {} bytes", data.len());
        // In reality: compress the data
    }
}

impl Plugin for EncryptionPlugin {
    fn name(&self) -> &str { "encrypt" }
    fn version(&self) -> &str { "2.1.0" }
    fn process(&self, data: &mut Vec<u8>) {
        println!("  [encrypt] Encrypting {} bytes with key '{}'", data.len(), self.key);
        // In reality: encrypt the data
    }
}

struct Pipeline {
    plugins: Vec<Box<dyn Plugin>>,
}

impl Pipeline {
    fn new() -> Self {
        Pipeline { plugins: Vec::new() }
    }

    fn add_plugin(&mut self, plugin: Box<dyn Plugin>) {
        println!("Loaded plugin: {} v{}", plugin.name(), plugin.version());
        self.plugins.push(plugin);
    }

    fn run(&self, data: &mut Vec<u8>) {
        println!("Running pipeline on {} bytes...", data.len());
        for plugin in &self.plugins {
            plugin.process(data);
        }
        println!("Pipeline complete.");
    }
}

fn main() {
    let mut pipeline = Pipeline::new();
    pipeline.add_plugin(Box::new(CompressionPlugin));
    pipeline.add_plugin(Box::new(EncryptionPlugin {
        key: "s3cret".into(),
    }));

    let mut data = vec![1, 2, 3, 4, 5];
    pipeline.run(&mut data);
}
```

---

## 11. The `impl Trait` Return Type

### Returning a Single Concrete Type

`impl Trait` in return position means "I'm returning **one specific type** that
implements this trait, but I'm not telling you which one":

```rust
fn make_greeter() -> impl Fn(&str) -> String {
    |name| format!("Hello, {}!", name)
}
```

This is **static dispatch** — the compiler knows the exact closure type. But the
caller only sees the `Fn` interface.

### The Limitation: One Type Only

You **cannot** return different concrete types from different branches:

```rust
fn make_shape(circle: bool) -> impl Shape {
    if circle {
        Circle { radius: 1.0 }     // type A
    } else {
        Rectangle { width: 1.0, height: 1.0 } // type B — ERROR!
    }
}
// ERROR: `if` and `else` have incompatible types
```

### The Solution: `Box<dyn Trait>`

When you need to return different types, use a boxed trait object:

```rust
fn make_shape(circle: bool) -> Box<dyn Shape> {
    if circle {
        Box::new(Circle { radius: 1.0 })
    } else {
        Box::new(Rectangle { width: 1.0, height: 1.0 })
    }
}
```

### Comparison Table

| Feature                     | `-> impl Trait`              | `-> Box<dyn Trait>`              |
|-----------------------------|------------------------------|----------------------------------|
| Number of possible types    | Exactly one                  | Any number                       |
| Dispatch                    | Static                       | Dynamic                          |
| Heap allocation             | No (unless the type itself allocates) | Yes (the `Box`)         |
| Caller knows concrete type? | No (but compiler does)       | No                               |
| Use case                    | Closures, iterators, builders| Heterogeneous returns, factories |

### `impl Trait` in Iterators

One of the most common uses — hiding complex iterator types:

```rust
fn even_squares(limit: u64) -> impl Iterator<Item = u64> {
    (0..limit)
        .map(|x| x * x)
        .filter(|x| x % 2 == 0)
}

fn main() {
    for n in even_squares(20) {
        print!("{} ", n);
    }
    // Output: 0 4 16 36 64 100 144 196 256 324
}
```

Without `impl Iterator`, you'd have to write the full type:
`std::iter::Filter<std::iter::Map<std::ops::Range<u64>, ...>, ...>` — a
nightmare.

---

## 12. Exercises

### Exercise 1: Static and Dynamic Side by Side

Create a trait `Summary` with a method `fn summarize(&self) -> String`.
Implement it for `Article` (title + author) and `Tweet` (username + content,
max 50 chars).

1. Write a function `print_summary` using **static dispatch**.
2. Write a function `print_all_summaries` that takes a `&[Box<dyn Summary>]`
   and prints every summary using **dynamic dispatch**.
3. In `main`, demonstrate both.

<details>
<summary><strong>Solution</strong></summary>

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article {
    title: String,
    author: String,
}

struct Tweet {
    username: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{} by {}", self.title, self.author)
    }
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        let preview = if self.content.len() > 50 {
            format!("{}...", &self.content[..50])
        } else {
            self.content.clone()
        };
        format!("@{}: {}", self.username, preview)
    }
}

// Static dispatch
fn print_summary(item: &impl Summary) {
    println!("Summary: {}", item.summarize());
}

// Dynamic dispatch
fn print_all_summaries(items: &[Box<dyn Summary>]) {
    for item in items {
        println!("Summary: {}", item.summarize());
    }
}

fn main() {
    let article = Article {
        title: "Rust 2026 Edition".into(),
        author: "The Rust Team".into(),
    };
    let tweet = Tweet {
        username: "rustlang".into(),
        content: "Excited about the new edition!".into(),
    };

    // Static dispatch
    print_summary(&article);
    print_summary(&tweet);

    // Dynamic dispatch
    let items: Vec<Box<dyn Summary>> = vec![
        Box::new(Article {
            title: "Async Traits Stabilized".into(),
            author: "Niko".into(),
        }),
        Box::new(Tweet {
            username: "ferris".into(),
            content: "🦀🦀🦀".into(),
        }),
    ];
    print_all_summaries(&items);
}
```

</details>

---

### Exercise 2: Object Safety Detective

Which of these traits are object-safe? For each one that isn't, explain why
and fix it.

```rust
// Trait A
trait Formatter {
    fn format(&self) -> String;
}

// Trait B
trait Constructible {
    fn new(value: i32) -> Self;
}

// Trait C
trait Encoder {
    fn encode<W: std::io::Write>(&self, writer: &mut W) -> std::io::Result<()>;
}

// Trait D
trait Drawable {
    fn draw(&self);
    fn clone_box(&self) -> Box<dyn Drawable>;
}
```

<details>
<summary><strong>Solution</strong></summary>

```rust
// Trait A — ✅ Object-safe
// All methods take &self and return a fixed-size type.
trait Formatter {
    fn format(&self) -> String;
}

// Trait B — ❌ NOT object-safe
// `new` returns `Self` and has no `self` parameter (associated function).
// Fix: add `where Self: Sized` to exclude it from the vtable.
trait Constructible {
    fn new(value: i32) -> Self
    where
        Self: Sized;
}

// Trait C — ❌ NOT object-safe
// `encode` has a generic type parameter `W`.
// Fix: use a trait object instead of a generic.
trait Encoder {
    fn encode(&self, writer: &mut dyn std::io::Write) -> std::io::Result<()>;
}

// Trait D — ✅ Object-safe (already!)
// `clone_box` returns Box<dyn Drawable>, not `Self`.
// `draw` takes `&self`. Both are fine.
trait Drawable {
    fn draw(&self);
    fn clone_box(&self) -> Box<dyn Drawable>;
}
```

</details>

---

### Exercise 3: Build a Mini Calculator with Plugins

Design a calculator where operations are plugins:

1. Define a trait `Operation` with methods `fn name(&self) -> &str` and
   `fn compute(&self, a: f64, b: f64) -> f64`.
2. Implement `Add`, `Multiply`, and `Power` structs.
3. Create a `Calculator` struct that holds `Vec<Box<dyn Operation>>`.
4. Add a method `run(&self, op_name: &str, a: f64, b: f64) -> Option<f64>`
   that finds the operation by name and computes the result.
5. In `main`, register all three operations and run a few computations.

<details>
<summary><strong>Solution</strong></summary>

```rust
trait Operation {
    fn name(&self) -> &str;
    fn compute(&self, a: f64, b: f64) -> f64;
}

struct Add;
struct Multiply;
struct Power;

impl Operation for Add {
    fn name(&self) -> &str { "add" }
    fn compute(&self, a: f64, b: f64) -> f64 { a + b }
}

impl Operation for Multiply {
    fn name(&self) -> &str { "multiply" }
    fn compute(&self, a: f64, b: f64) -> f64 { a * b }
}

impl Operation for Power {
    fn name(&self) -> &str { "power" }
    fn compute(&self, a: f64, b: f64) -> f64 { a.powf(b) }
}

struct Calculator {
    operations: Vec<Box<dyn Operation>>,
}

impl Calculator {
    fn new() -> Self {
        Calculator { operations: Vec::new() }
    }

    fn register(&mut self, op: Box<dyn Operation>) {
        self.operations.push(op);
    }

    fn run(&self, op_name: &str, a: f64, b: f64) -> Option<f64> {
        self.operations
            .iter()
            .find(|op| op.name() == op_name)
            .map(|op| op.compute(a, b))
    }
}

fn main() {
    let mut calc = Calculator::new();
    calc.register(Box::new(Add));
    calc.register(Box::new(Multiply));
    calc.register(Box::new(Power));

    let tests = [
        ("add", 3.0, 4.0),
        ("multiply", 6.0, 7.0),
        ("power", 2.0, 10.0),
        ("subtract", 5.0, 3.0),  // not registered
    ];

    for (op, a, b) in &tests {
        match calc.run(op, *a, *b) {
            Some(result) => println!("{op}({a}, {b}) = {result}"),
            None => println!("{op}: unknown operation"),
        }
    }
    // Output:
    // add(3, 4) = 7
    // multiply(6, 7) = 42
    // power(2, 10) = 1024
    // subtract: unknown operation
}
```

</details>

---

## 13. Summary

| Concept                        | Key Point                                                        |
|--------------------------------|------------------------------------------------------------------|
| **Dispatch**                   | Choosing which function implementation to call                   |
| **Static dispatch**            | Resolved at compile time via monomorphization; zero runtime cost |
| **Dynamic dispatch**           | Resolved at runtime via vtable; enables heterogeneous collections|
| **Vtable**                     | Table of function pointers (+ drop, size, align) per (Type, Trait) pair |
| **`&dyn Trait`**               | Fat pointer: data pointer + vtable pointer                       |
| **`impl Trait` (argument)**    | Syntactic sugar for generics — static dispatch                   |
| **`impl Trait` (return)**      | Hides one concrete type — static dispatch                        |
| **`Box<dyn Trait>` (return)**  | Can return different concrete types — dynamic dispatch           |
| **Object safety**              | Trait must have no `Self` returns, no generic methods, no `Sized` bound |
| **Enum dispatch**              | Alternative: closed set of types, no vtable, no code duplication |
| **Default choice**             | Start with static; reach for dynamic when you need flexibility   |

---

| [⬅ Previous: 08-supertraits](08-supertraits.md) | [Next Stage: Lifetimes ➡](../09-lifetimes/README.md) |
|:-------------------------------------------------|------------------------------------------------------:|

**Tutorial 9 of 9 — Stage 8: Generics & Traits**

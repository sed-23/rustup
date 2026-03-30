# Defining Traits — Rust's Interfaces 🎯

> **"A trait defines shared behavior: a contract that types promise to fulfill."**

---

## Table of Contents

- [What Is a Trait?](#what-is-a-trait)
- [Defining a Trait](#defining-a-trait)
- [Required vs Default Methods](#required-vs-default-methods)
- [Default Method Implementations](#default-method-implementations)
- [Self and self in Traits](#self-and-self-in-traits)
- [Associated Types](#associated-types)
- [Associated Constants](#associated-constants)
- [Traits in Other Languages](#traits-in-other-languages)
- [The History of Traits](#the-history-of-traits)
- [Trait Design Guidelines](#trait-design-guidelines)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is a Trait?

A **trait** defines shared behavior. It's a set of methods (and sometimes types
or constants) that a type *promises* to implement. When a type implements a
trait, it's making a guarantee: "I can do this thing."

Think of a trait as a **capability contract**:

```rust
// Any type that implements `Greet` promises it can say hello
trait Greet {
    fn hello(&self) -> String;
}
```

If you're coming from another language, a trait is similar to:

| You know…              | Rust equivalent |
|------------------------|-----------------|
| Java **interface**     | `trait`         |
| Swift **protocol**     | `trait`         |
| Haskell **typeclass**  | `trait`         |
| C++ **abstract class** | `trait`         |
| Go **interface**       | `trait`         |
| Python **ABC**         | `trait`         |

But Rust traits are more powerful than most of these. They support default
implementations, associated types, associated constants, and — critically —
they are the foundation of Rust's entire generic system. Every time you write
a generic bound like `T: Clone`, you're using a trait.

### Why traits matter

Without traits, every type is an island. Traits let you:

1. **Write generic code** — functions that work on *any* type with a given capability
2. **Define shared behavior** — without inheritance hierarchies
3. **Enable polymorphism** — both static (monomorphization) and dynamic (`dyn Trait`)
4. **Compose capabilities** — a type can implement as many traits as it needs

```
┌─────────────────────────────────────────────────┐
│                    trait Summary                 │
│  ┌───────────────────────────────────────────┐  │
│  │  fn summarize(&self) -> String;           │  │  ← required method
│  ├───────────────────────────────────────────┤  │
│  │  fn preview(&self) -> String {            │  │
│  │      format!("{}...", &self.summarize()   │  │  ← default method
│  │          [..50])                          │  │
│  │  }                                        │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
│   Implementors MUST provide: summarize()        │
│   Implementors CAN override:  preview()         │
└─────────────────────────────────────────────────┘
```

---

## Defining a Trait

The syntax for defining a trait is straightforward:

```rust
trait TraitName {
    // method signatures and/or default implementations
}
```

Here's a real example:

```rust
trait Summary {
    fn summarize(&self) -> String;
}
```

That's it. One line inside the braces, and we've defined a contract: any type
that implements `Summary` must provide a `summarize` method that takes `&self`
and returns a `String`.

### A more complete trait

```rust
trait Summary {
    /// Returns the author of this content.
    fn author(&self) -> &str;

    /// Returns a one-line summary.
    fn summarize(&self) -> String;

    /// Returns a short preview (default: uses summarize).
    fn preview(&self) -> String {
        format!("{}...", &self.summarize()[..50.min(self.summarize().len())])
    }
}
```

This trait has:
- Two **required methods**: `author()` and `summarize()`
- One **default method**: `preview()` — which has a body and can be used as-is

### Visibility

Traits follow the same visibility rules as other Rust items:

```rust
pub trait Summary {          // accessible outside this module
    fn summarize(&self) -> String;
}

trait InternalHelper {       // private to this module
    fn helper(&self) -> u32;
}
```

A `pub` trait can be implemented by external crates (unless you use the
sealed-trait pattern, which we'll cover later in this stage).

---

## Required vs Default Methods

Every method in a trait is either **required** or **default**:

```rust
trait Drawable {
    // REQUIRED — no body, just a signature ending with `;`
    fn draw(&self);

    // REQUIRED — still no body
    fn bounding_box(&self) -> (f64, f64, f64, f64);

    // DEFAULT — has a body (implementors get this for free)
    fn is_visible(&self) -> bool {
        true
    }

    // DEFAULT — also has a body
    fn draw_if_visible(&self) {
        if self.is_visible() {
            self.draw();
        }
    }
}
```

### Required methods

A required method has **no body** — just a signature followed by a semicolon:

```rust
fn draw(&self);
```

Any type that implements `Drawable` **must** provide an implementation of
`draw()` and `bounding_box()`. If it doesn't, the compiler will refuse to
compile:

```
error[E0046]: not all trait items implemented, missing: `draw`
 --> src/main.rs:12:1
  |
  = note: `draw` from trait `Drawable` is not implemented
```

### Default methods

A default method **has a body**:

```rust
fn is_visible(&self) -> bool {
    true
}
```

Implementors can:
- **Use the default** — just don't mention the method at all
- **Override it** — provide their own implementation

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
    visible: bool,
}

impl Drawable for Circle {
    // MUST provide required methods
    fn draw(&self) {
        println!("Drawing circle at ({}, {}) r={}", self.x, self.y, self.radius);
    }

    fn bounding_box(&self) -> (f64, f64, f64, f64) {
        (
            self.x - self.radius,
            self.y - self.radius,
            self.x + self.radius,
            self.y + self.radius,
        )
    }

    // Override the default
    fn is_visible(&self) -> bool {
        self.visible
    }

    // draw_if_visible() — we get this for free, no need to implement it
}
```

---

## Default Method Implementations

Default methods are one of Rust's most useful trait features. They let you
provide "batteries included" behavior while still allowing customization.

### Basic default

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

Any type that implements `Summary` without overriding `summarize()` gets
`"(Read more...)"` automatically.

### Default methods calling required methods

This is where default methods become truly powerful. A default method can call
**other trait methods** — even required ones that don't have a body yet:

```rust
trait Summary {
    // Required — every implementor must provide these
    fn title(&self) -> &str;
    fn author(&self) -> &str;

    // Default — calls the required methods above!
    fn summarize(&self) -> String {
        format!("{} by {}", self.title(), self.author())
    }
}
```

The compiler knows that by the time `summarize()` is called, the implementing
type *will* have provided `title()` and `author()`. So we can safely call them
in the default body.

```rust
struct BlogPost {
    title: String,
    author: String,
    content: String,
}

impl Summary for BlogPost {
    fn title(&self) -> &str {
        &self.title
    }

    fn author(&self) -> &str {
        &self.author
    }

    // summarize() is free — it uses the default that calls title() and author()
}

fn main() {
    let post = BlogPost {
        title: String::from("Rust Traits Deep Dive"),
        author: String::from("Alice"),
        content: String::from("Traits are amazing..."),
    };

    // Uses the default summarize() which calls our title() and author()
    println!("{}", post.summarize());
    // Output: "Rust Traits Deep Dive by Alice"
}
```

### Default methods calling other default methods

Default methods can also call other default methods:

```rust
trait Formatter {
    fn name(&self) -> &str;

    fn greeting(&self) -> String {
        format!("Hello, {}!", self.name())
    }

    fn farewell(&self) -> String {
        format!("Goodbye, {}!", self.name())
    }

    fn full_conversation(&self) -> String {
        format!("{}\n...\n{}", self.greeting(), self.farewell())
    }
}
```

Here `full_conversation()` calls `greeting()` and `farewell()`, which each call
`name()`. All four methods form a chain, and an implementor only *needs* to
provide `name()`.

### A warning about overriding

If a type overrides a default method, it **cannot** call the original default.
There's no `super.method()` in Rust traits:

```rust
trait Logger {
    fn log(&self, msg: &str) {
        println!("[LOG] {}", msg);
    }
}

struct FileLogger;

impl Logger for FileLogger {
    fn log(&self, msg: &str) {
        // ❌ Cannot call the default Logger::log() from here
        // There is no `super.log(msg)` syntax
        println!("[FILE] {}", msg);
    }
}
```

This is a deliberate design choice. If you need layered behavior, compose it
differently — often with helper functions or wrapper types.

---

## Self and self in Traits

Trait methods use `self` in several forms. Understanding the distinction between
lowercase `self` and uppercase `Self` is critical.

### `&self` — immutable borrow

The most common. The method borrows the value but cannot modify it:

```rust
trait Describe {
    fn describe(&self) -> String;
}
```

This is sugar for `fn describe(self: &Self) -> String;`.

### `&mut self` — mutable borrow

The method borrows the value and *can* modify it:

```rust
trait Counter {
    fn increment(&mut self);
    fn count(&self) -> u32;
}
```

### `self` — takes ownership (consuming)

The method **takes ownership** of the value. After calling it, the original
value is moved and can no longer be used:

```rust
trait IntoMessage {
    fn into_message(self) -> String;
}
```

This is useful for conversions or when the method needs to destructure the
value. The standard library's `Into` trait uses this pattern:

```rust
// from std:
pub trait Into<T> {
    fn into(self) -> T;
}
```

### `Self` — the implementing type

Uppercase `Self` is a **type alias** for whatever type is implementing the
trait. It's like a placeholder:

```rust
trait Clonable {
    fn duplicate(&self) -> Self;
}

struct Point {
    x: f64,
    y: f64,
}

impl Clonable for Point {
    fn duplicate(&self) -> Self {
        // Here, `Self` is `Point`
        Point {
            x: self.x,
            y: self.y,
        }
    }
}
```

### `Self` in return position

`Self` is often used as a return type:

```rust
trait Builder {
    fn new() -> Self;
    fn build(self) -> Self;
}
```

### Summary table

| Syntax       | Meaning                        | Ownership          |
|--------------|--------------------------------|--------------------|
| `&self`      | Immutable borrow               | Borrowed           |
| `&mut self`  | Mutable borrow                 | Borrowed (mutable) |
| `self`       | Takes ownership                | Moved              |
| `Self`       | The implementing type          | (It's a type)      |

### All four in one trait

```rust
trait Transform {
    /// Inspect without modifying
    fn inspect(&self) -> String;

    /// Modify in place
    fn scale(&mut self, factor: f64);

    /// Consume and produce a new value
    fn into_scaled(self, factor: f64) -> Self;

    /// Associated function (no self at all) — like a static method
    fn origin() -> Self;
}
```

Note the last method, `origin()`, which has **no** `self` parameter at all.
This is an **associated function** — called as `Type::origin()` rather than
`value.origin()`. You've already seen this pattern with `String::new()`.

---

## Associated Types

An **associated type** is a type placeholder defined inside a trait. Instead of
making the trait generic, you let each implementor choose the concrete type.

### Syntax

```rust
trait Iterator {
    type Item;                                    // associated type

    fn next(&mut self) -> Option<Self::Item>;     // uses it
}
```

The `type Item;` declaration says: "Every type that implements `Iterator` must
specify what `Item` is."

### Example: implementing Iterator

```rust
struct Counter {
    count: u32,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Self {
        Counter { count: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;                    // Item is u32 for this type

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    let counter = Counter::new(3);
    let items: Vec<u32> = counter.collect();
    println!("{:?}", items); // [1, 2, 3]
}
```

### Why associated types instead of generic parameters?

You might wonder: why not just make the trait generic?

```rust
// Generic parameter approach
trait Iterator<Item> {
    fn next(&mut self) -> Option<Item>;
}

// Associated type approach
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

The key difference: **a type can implement a generic trait multiple times** (once
for each type parameter), but a type can implement a trait with an associated
type **only once**.

```rust
// With generics: a type could implement Iterator<u32> AND Iterator<String>
// Which one does `next()` return? Ambiguous!

// With associated types: a type implements Iterator once, Item is fixed
// No ambiguity — Counter's Item is always u32
```

**Rule of thumb:**
- Use **associated types** when each implementor should have exactly *one*
  natural choice (like `Iterator::Item`)
- Use **generic parameters** when a type might reasonably implement the trait
  for multiple types (like `From<T>` — a type can convert from many sources)

### Defining your own trait with associated types

```rust
trait Graph {
    type Node;
    type Edge;

    fn nodes(&self) -> Vec<&Self::Node>;
    fn edges(&self) -> Vec<&Self::Edge>;
    fn neighbors(&self, node: &Self::Node) -> Vec<&Self::Node>;
}
```

Each graph implementation specifies its own `Node` and `Edge` types. A
`SocialGraph` might use `Person` and `Friendship`, while a `CityMap` might
use `Intersection` and `Road`.

### Associated types with bounds

You can constrain associated types:

```rust
trait Container {
    type Item: std::fmt::Display + Clone;   // Item must be Display + Clone

    fn items(&self) -> &[Self::Item];

    fn print_all(&self) {
        for item in self.items() {
            println!("{}", item);           // works because Item: Display
        }
    }
}
```

---

## Associated Constants

Traits can also define **associated constants** — constant values that each
implementor provides or that have defaults.

### Syntax

```rust
trait Bounded {
    const MIN: Self;
    const MAX: Self;
}
```

### Example

```rust
trait Identifiable {
    const TYPE_NAME: &'static str;
    const VERSION: u32 = 1;           // default value

    fn id(&self) -> String;
}

struct User {
    name: String,
}

impl Identifiable for User {
    const TYPE_NAME: &'static str = "User";
    // VERSION uses the default (1)

    fn id(&self) -> String {
        format!("user_{}", self.name.to_lowercase())
    }
}

struct Admin {
    name: String,
    level: u8,
}

impl Identifiable for Admin {
    const TYPE_NAME: &'static str = "Admin";
    const VERSION: u32 = 2;            // override the default

    fn id(&self) -> String {
        format!("admin_L{}_{}", self.level, self.name.to_lowercase())
    }
}

fn print_info<T: Identifiable>(item: &T) {
    println!("[{} v{}] {}", T::TYPE_NAME, T::VERSION, item.id());
}

fn main() {
    let user = User { name: String::from("Alice") };
    let admin = Admin { name: String::from("Bob"), level: 3 };

    print_info(&user);   // [User v1] user_alice
    print_info(&admin);  // [Admin v2] admin_L3_bob
}
```

### When to use associated constants

- **Configuration values** — like buffer sizes, version numbers, or names
- **Mathematical traits** — zero, one, min, max for numeric types
- **Protocol definitions** — magic bytes, header sizes

```rust
trait Packet {
    const HEADER_SIZE: usize;
    const MAX_PAYLOAD: usize;
    const MAGIC_BYTES: [u8; 4];

    fn encode(&self) -> Vec<u8>;
    fn decode(data: &[u8]) -> Option<Self> where Self: Sized;
}
```

---

## Traits in Other Languages

Rust's traits aren't invented from nothing. They have relatives across many
languages. Understanding the similarities and differences helps you build
intuition.

### Java interfaces

Java interfaces are the closest mainstream analogy. Since **Java 8**, they
support default methods too:

```java
// Java
interface Summary {
    String summarize();                          // required (abstract)

    default String preview() {                   // default method (Java 8+)
        return summarize().substring(0, 50) + "...";
    }
}

class Article implements Summary {
    public String summarize() {
        return "Article content here...";
    }
    // preview() is inherited
}
```

**Differences from Rust:**
- Java interfaces can't have associated types (until limited support via generics)
- Java uses class-based inheritance; Rust has no classes
- Java allows interface inheritance (`extends`); Rust uses supertraits

### C++ abstract classes

```cpp
// C++
class Summary {
public:
    virtual std::string summarize() = 0;   // pure virtual (required)

    virtual std::string preview() {         // virtual with default
        return summarize().substr(0, 50) + "...";
    }

    virtual ~Summary() = default;
};
```

**Differences from Rust:**
- C++ abstract classes are tied to the inheritance hierarchy
- C++ uses vtables for all virtual dispatch; Rust uses monomorphization by default
- C++ has no associated types in abstract classes
- A C++ class can only inherit from one class (without multiple inheritance headaches)

### Haskell typeclasses

Haskell typeclasses are the **direct inspiration** for Rust traits:

```haskell
-- Haskell
class Summary a where
    summarize :: a -> String

    preview :: a -> String        -- default implementation
    preview x = take 50 (summarize x) ++ "..."
```

**Similarities:**
- Ad hoc polymorphism (different behavior per type)
- Default implementations
- Type constraints (bounds)

**Differences:**
- Haskell typeclasses work with type inference more deeply
- Haskell has higher-kinded types (Rust doesn't, yet)
- Haskell separates data and behavior entirely; Rust attaches `impl` blocks to structs

### Go interfaces

Go's interfaces are **structurally typed** — you don't explicitly say "I implement
this interface." If your type has the right methods, it just works:

```go
// Go
type Summary interface {
    Summarize() string
}

// No "implements" keyword — Duck satisfies Summary automatically
type Article struct { Content string }

func (a Article) Summarize() string {
    return a.Content
}
```

**Differences from Rust:**
- Go: implicit implementation (structural typing)
- Rust: explicit implementation (`impl Trait for Type`)
- Go interfaces have no default methods
- Go interfaces have no associated types or constants

### Swift protocols

Swift protocols are remarkably similar to Rust traits:

```swift
// Swift
protocol Summary {
    var author: String { get }        // required property
    func summarize() -> String        // required method
}

extension Summary {
    func preview() -> String {        // default implementation (via extension)
        return String(summarize().prefix(50)) + "..."
    }
}
```

**Similarities:**
- Explicit conformance (`struct Article: Summary`)
- Associated types (`associatedtype Item`)
- Protocol extensions (like Rust's default methods)

**Differences:**
- Swift has protocol inheritance and class-bound protocols
- Swift's protocol witness tables differ from Rust's monomorphization

### Python ABCs

Python Abstract Base Classes exist but are rarely used because Python prefers
duck typing:

```python
# Python
from abc import ABC, abstractmethod

class Summary(ABC):
    @abstractmethod
    def summarize(self) -> str:
        pass

    def preview(self) -> str:          # default
        return self.summarize()[:50] + "..."
```

**Differences:**
- Python: no compile-time enforcement (only runtime errors)
- Python: duck typing makes ABCs optional
- Python: no associated types or constants in ABCs

### Comparison table

| Language | Concept          | Explicit impl? | Default Methods? | Associated Types? | Multiple Impl? |
|----------|------------------|:--------------:|:----------------:|:-----------------:|:--------------:|
| Rust     | Trait            | ✅ Yes          | ✅ Yes            | ✅ Yes             | ✅ Yes          |
| Java     | Interface        | ✅ Yes          | ✅ Yes (8+)       | ❌ No              | ✅ Yes          |
| C++      | Abstract Class   | ✅ Yes          | ✅ Yes            | ❌ No              | ⚠️ Complicated  |
| Haskell  | Typeclass        | ✅ Yes          | ✅ Yes            | ✅ Yes             | ✅ Yes          |
| Go       | Interface        | ❌ No           | ❌ No             | ❌ No              | ✅ Yes          |
| Swift    | Protocol         | ✅ Yes          | ✅ Yes            | ✅ Yes             | ✅ Yes          |
| Python   | ABC              | ✅ Yes          | ✅ Yes            | ❌ No              | ✅ Yes          |

> **"Multiple Impl"** here means: can a single type implement/conform to multiple
> traits/interfaces/protocols simultaneously?

---

## The History of Traits

Understanding where traits came from helps you appreciate their design.

### Timeline

```
1987 — Self language introduces "traits" as collections of methods
       (David Ungar & Randall B. Smith, Xerox PARC)

1989 — Haskell 1.0 introduces "typeclasses"
       (Philip Wadler & Stephen Blott)
       → Ad hoc polymorphism without OOP class hierarchies

2003 — Schärli et al. publish "Traits: Composable Units of Behaviour"
       → Formalizes traits as a composition mechanism for OOP

2004 — Scala launches with traits (mix-in composition)

2010 — Rust development begins (Graydon Hoare at Mozilla)
       Early Rust experiments with typeclasses and interfaces

2012 — PHP 5.4 adds traits (horizontal code reuse)

2013 — Rust settles on "trait" as the keyword
       Combines Haskell typeclass ideas with OOP interface ergonomics

2014 — Java 8 adds default methods to interfaces

2015 — Rust 1.0 released with traits as the core abstraction mechanism

2024 — Rust traits continue to evolve (async traits, return-position
       impl Trait in traits, trait aliases RFC)
```

### Key insight

Rust's traits took the **best ideas from two worlds**:

1. **From Haskell typeclasses:** Ad hoc polymorphism, trait bounds, associated
   types, coherence rules (orphan rule)
2. **From OOP interfaces:** Familiar syntax, method dispatch on `self`,
   default implementations

The result is a system that's more powerful than Java interfaces, safer than
C++ abstract classes, and more practical than Haskell typeclasses for systems
programming.

---

## Trait Design Guidelines

Designing good traits is an art. Here are guidelines the Rust community has
developed over the years.

### 1. Keep traits focused

Each trait should represent **one capability** — the Single Responsibility
Principle applied to traits:

```rust
// ✅ Good: focused traits
trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
}

trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
}

// ❌ Bad: kitchen-sink trait
trait ReadWriteSeekCloseFlush {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn seek(&mut self, pos: u64) -> Result<u64>;
    fn close(self) -> Result<()>;
    fn flush(&mut self) -> Result<()>;
}
```

### 2. Prefer many small traits

Small traits are easier to implement, compose, and reason about. Types can
implement multiple traits, so there's no cost:

```rust
trait Saveable {
    fn save(&self) -> Result<(), Error>;
}

trait Loadable {
    fn load(path: &str) -> Result<Self, Error> where Self: Sized;
}

trait Deletable {
    fn delete(&self) -> Result<(), Error>;
}

// A type picks the capabilities it needs
struct Document { /* ... */ }
impl Saveable for Document { /* ... */ }
impl Loadable for Document { /* ... */ }
// Document is NOT Deletable — and that's fine
```

### 3. Use default methods wisely

Default methods reduce boilerplate for implementors. Use them when there's an
**obvious, correct default** that most types would want:

```rust
trait Logger {
    fn log(&self, level: &str, message: &str);

    // Most loggers want these convenience methods
    fn info(&self, msg: &str) {
        self.log("INFO", msg);
    }

    fn warn(&self, msg: &str) {
        self.log("WARN", msg);
    }

    fn error(&self, msg: &str) {
        self.log("ERROR", msg);
    }
}
```

Implementors only need to provide `log()`. They get `info()`, `warn()`, and
`error()` for free.

### 4. Naming conventions

Rust traits typically use **adjectives** or **nouns**:

| Pattern       | Examples                                    |
|---------------|---------------------------------------------|
| **-able**     | `Readable`, `Drawable`, `Comparable`        |
| **-ible**     | `Convertible`, `Accessible`                 |
| **-er/-or**   | `Iterator`, `Visitor`, `Handler`, `Reader`  |
| **Noun**      | `Display`, `Debug`, `Hash`, `Clone`, `Copy` |
| **Into/From** | `Into<T>`, `From<T>`, `TryFrom<T>`         |

Avoid:
- Verbs as trait names (`trait Read` is fine — it's also a noun)
- `I`-prefix (`IReadable`) — that's a C#/Java convention, not Rust
- `Trait` suffix (`SummaryTrait`) — just `Summary`

### 5. Document the contract

Use doc comments to explain:
- What the trait represents
- What implementors must guarantee (invariants)
- How default methods behave

```rust
/// A type that can be serialized to a byte sequence.
///
/// # Contract
///
/// - `serialize()` must produce output that `deserialize()` can reconstruct
/// - Serialization must be deterministic (same input → same output)
/// - Serialization must not panic for valid instances
trait Serialize {
    /// Converts this value to a byte vector.
    fn serialize(&self) -> Vec<u8>;
}
```

### 6. Consider object safety

If you need dynamic dispatch (`dyn Trait`), your trait must be **object-safe**.
We'll cover this in detail later in this stage, but the short version: avoid
methods that return `Self` or use generic type parameters if you want `dyn`.

---

## Exercises

### Exercise 1: Define a `Shape` trait

Define a trait called `Shape` with:
- A required method `area(&self) -> f64`
- A required method `perimeter(&self) -> f64`
- A default method `describe(&self) -> String` that returns
  `"Shape with area {area} and perimeter {perimeter}"`

Then implement `Shape` for `Circle` and `Rectangle`:

```rust
struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}
```

<details>
<summary>💡 Solution</summary>

```rust
use std::f64::consts::PI;

trait Shape {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;

    fn describe(&self) -> String {
        format!(
            "Shape with area {:.2} and perimeter {:.2}",
            self.area(),
            self.perimeter()
        )
    }
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }

    fn perimeter(&self) -> f64 {
        2.0 * PI * self.radius
    }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }
}

fn main() {
    let circle = Circle { radius: 5.0 };
    let rect = Rectangle { width: 4.0, height: 6.0 };

    println!("{}", circle.describe());
    // Shape with area 78.54 and perimeter 31.42

    println!("{}", rect.describe());
    // Shape with area 24.00 and perimeter 20.00
}
```

</details>

---

### Exercise 2: Trait with associated type and constant

Define a trait called `Codec` with:
- An associated type `Output`
- An associated constant `NAME: &'static str`
- A required method `encode(&self) -> Self::Output`
- A default method `info(&self) -> String` that returns `"Codec: {NAME}"`

Implement it for a `JsonCodec` struct (where `Output = String`) and a
`BinaryCodec` struct (where `Output = Vec<u8>`).

<details>
<summary>💡 Solution</summary>

```rust
trait Codec {
    type Output;
    const NAME: &'static str;

    fn encode(&self, data: &str) -> Self::Output;

    fn info(&self) -> String {
        format!("Codec: {}", Self::NAME)
    }
}

struct JsonCodec;
struct BinaryCodec;

impl Codec for JsonCodec {
    type Output = String;
    const NAME: &'static str = "JSON";

    fn encode(&self, data: &str) -> Self::Output {
        format!("{{\"data\": \"{}\"}}", data)
    }
}

impl Codec for BinaryCodec {
    type Output = Vec<u8>;
    const NAME: &'static str = "Binary";

    fn encode(&self, data: &str) -> Self::Output {
        data.as_bytes().to_vec()
    }
}

fn main() {
    let json = JsonCodec;
    let binary = BinaryCodec;

    println!("{}", json.info());             // Codec: JSON
    println!("{}", json.encode("hello"));    // {"data": "hello"}

    println!("{}", binary.info());           // Codec: Binary
    println!("{:?}", binary.encode("hello")); // [104, 101, 108, 108, 111]
}
```

</details>

---

### Exercise 3: The four flavors of `self`

Define a trait called `Token` that uses all four forms of `self`/`Self`:
- `fn value(&self) -> &str` — borrow to inspect
- `fn refresh(&mut self)` — mutably borrow to modify
- `fn consume(self) -> String` — take ownership
- `fn generate() -> Self where Self: Sized` — associated function returning `Self`

Implement it for an `ApiToken` struct that wraps a `String`.

<details>
<summary>💡 Solution</summary>

```rust
trait Token {
    fn value(&self) -> &str;
    fn refresh(&mut self);
    fn consume(self) -> String;
    fn generate() -> Self where Self: Sized;
}

struct ApiToken {
    token: String,
}

impl Token for ApiToken {
    fn value(&self) -> &str {
        &self.token
    }

    fn refresh(&mut self) {
        // In a real app, this would call an API
        self.token = format!("refreshed_{}", &self.token[..8.min(self.token.len())]);
    }

    fn consume(self) -> String {
        // Takes ownership — self is moved
        format!("CONSUMED: {}", self.token)
    }

    fn generate() -> Self {
        ApiToken {
            token: String::from("tok_abc123xyz"),
        }
    }
}

fn main() {
    let mut token = ApiToken::generate();    // associated function (no self)
    println!("Value: {}", token.value());    // &self — borrow

    token.refresh();                          // &mut self — mutable borrow
    println!("Refreshed: {}", token.value());

    let consumed = token.consume();           // self — ownership transferred
    println!("{}", consumed);

    // println!("{}", token.value());         // ❌ Would fail — token was moved
}
```

</details>

---

## Summary

| Concept                 | Syntax / Key Point                                           |
|-------------------------|--------------------------------------------------------------|
| **Define a trait**      | `trait Name { fn method(&self); }`                           |
| **Required method**     | No body — just signature + `;`                               |
| **Default method**      | Has a body — implementors get it for free                    |
| **Default calls required** | A default method can call required methods on `self`      |
| **`&self`**             | Immutable borrow                                             |
| **`&mut self`**         | Mutable borrow                                               |
| **`self`**              | Takes ownership (consuming)                                  |
| **`Self`**              | Type alias for the implementing type                         |
| **Associated type**     | `type Item;` — one concrete type per implementation          |
| **Associated constant** | `const MAX: usize;` — one value per implementation           |
| **Naming convention**   | Adjectives (`Readable`), nouns (`Iterator`), or Into/From    |
| **Design principle**    | Keep traits small, focused, and well-documented              |

### Key takeaways

1. A trait is a **contract** — a set of methods a type promises to provide
2. **Required methods** have no body; **default methods** do
3. Default methods can call other trait methods, even required ones
4. `self` controls **ownership**; `Self` is a **type alias**
5. **Associated types** fix one type per implementation (vs generics: many)
6. **Associated constants** let implementors provide configuration values
7. Rust traits draw from **Haskell typeclasses** and **OOP interfaces**
8. Design traits to be **small, focused, and composable**

---

**Previous:** [← Generic Structs & Enums](./02-generic-structs-enums.md) · **Next:** [Implementing Traits →](./04-implementing-traits.md)

<p align="center"><i>Tutorial 3 of 9 — Stage 8: Generics & Traits</i></p>

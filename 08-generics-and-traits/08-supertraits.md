# Supertraits & Trait Inheritance 🏗️

> **"Stand on the shoulders of traits — supertraits let you build powerful abstractions by requiring that implementors already satisfy other contracts."**

---

## Table of Contents

1. [What Are Supertraits?](#1-what-are-supertraits)
2. [Syntax](#2-syntax)
3. [Why Supertraits?](#3-why-supertraits)
4. [Example: OutlinePrint](#4-example-outlineprint)
5. [Standard Library Examples](#5-standard-library-examples)
6. [Supertraits vs OOP Inheritance](#6-supertraits-vs-oop-inheritance)
7. [Building Trait Hierarchies](#7-building-trait-hierarchies)
8. [Supertraits in Other Languages](#8-supertraits-in-other-languages)
9. [Associated Types with Supertraits](#9-associated-types-with-supertraits)
10. [Exercises](#10-exercises)
11. [Summary](#11-summary)

---

## 1. What Are Supertraits?

When you define a trait in Rust, you can **require** that any type implementing your trait
must *also* implement one or more other traits. Those required traits are called **supertraits**.

```rust
trait Printable: std::fmt::Display {
    fn print(&self) {
        println!("{}", self); // We can use `{}` because Display is guaranteed
    }
}
```

Here, `Display` is a **supertrait** of `Printable`. This means:

- Any type that implements `Printable` **must also** implement `Display`.
- Inside `Printable`'s default methods, you can freely call `Display` methods on `self`.
- The compiler enforces this — you'll get an error if you try to implement `Printable`
  for a type that doesn't implement `Display`.

Think of it as a **prerequisite**: "Before you can be `Printable`, you must first be `Display`."

```text
┌─────────────────────────────┐
│        Display              │  ← supertrait (prerequisite)
│   fn fmt(&self, ...) -> ... │
└──────────────┬──────────────┘
               │ requires
┌──────────────▼──────────────┐
│        Printable            │  ← subtrait (your trait)
│   fn print(&self)           │
└─────────────────────────────┘
```

---

## 2. Syntax

### Single Supertrait

```rust
trait SubTrait: SuperTrait {
    // methods...
}
```

### Multiple Supertraits

Use `+` to require more than one:

```rust
trait SubTrait: SuperTrait1 + SuperTrait2 {
    // methods...
}
```

### With Lifetime Bounds

You can mix in lifetime bounds too:

```rust
trait SubTrait: SuperTrait + 'static {
    // methods...
}
```

### With Where Clauses

For complex bounds, you can use a `where` clause:

```rust
trait SubTrait where Self: SuperTrait1 + SuperTrait2 {
    // methods...
}
```

This is equivalent to `trait SubTrait: SuperTrait1 + SuperTrait2` but can be
clearer when the bounds get long.

### Transitive Supertraits

Supertrait requirements are **transitive**:

```rust
trait A {
    fn a(&self);
}

trait B: A {
    fn b(&self);
}

trait C: B {
    fn c(&self);
}

// Implementing C requires implementing both B and A
```

If `C: B` and `B: A`, then implementing `C` requires implementing `B` **and** `A`.

---

## 3. Why Supertraits?

Supertraits solve a specific problem: **your trait's methods need to call methods
from another trait**.

### Without Supertraits (Doesn't Compile)

```rust
use std::fmt;

trait Printable {
    fn print(&self) {
        // ERROR: `Self` doesn't implement `Display`
        println!("{}", self);
    }
}
```

The compiler has no guarantee that `self` can be formatted with `{}`, so this fails.

### With Supertraits (Compiles)

```rust
use std::fmt;

trait Printable: fmt::Display {
    fn print(&self) {
        // OK: Display is guaranteed by the supertrait bound
        println!("{}", self);
    }
}
```

Now the compiler knows every `Printable` type is also `Display`, so `println!("{}", self)` is valid.

### Key Reasons to Use Supertraits

| Reason | Example |
|--------|---------|
| Default methods need other trait methods | `trait Printable: Display` — uses `{}` in default `print()` |
| Logical dependency between traits | `trait Ord: Eq + PartialOrd` — ordering requires equality |
| Composing capabilities | `trait Serialize: Debug + Clone` — serialization needs debug output and cloning |
| Enforcing a contract | `trait Error: Display + Debug` — errors must be displayable and debuggable |

---

## 4. Example: OutlinePrint

A classic example from *The Rust Programming Language*: a trait that prints a value
surrounded by a box of asterisks.

```rust
use std::fmt;

trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        // We can call to_string() because Display gives us that for free
        let output = self.to_string();
        let len = output.len();

        // Top border
        println!("{}", "*".repeat(len + 4));
        // Content with side borders
        println!("* {} *", output);
        // Bottom border
        println!("{}", "*".repeat(len + 4));
    }
}
```

### Implementing It

```rust
struct Point {
    x: i32,
    y: i32,
}

// Step 1: Implement the supertrait (Display)
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

// Step 2: Now we can implement OutlinePrint
impl OutlinePrint for Point {}

fn main() {
    let p = Point { x: 10, y: 20 };
    p.outline_print();
}
```

**Output:**

```text
**********
* (10, 20) *
**********
```

### What Happens If Display Is Missing?

```rust
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

// This won't compile!
impl OutlinePrint for Color {}
```

```text
error[E0277]: `Color` doesn't implement `std::fmt::Display`
  --> src/main.rs:XX:XX
   |
XX | impl OutlinePrint for Color {}
   |      ^^^^^^^^^^^^ `Color` cannot be formatted with the default formatter
   |
   = help: the trait `std::fmt::Display` is not implemented for `Color`
note: required by a bound in `OutlinePrint`
```

The compiler tells you *exactly* what's missing. Fix it by implementing `Display` for `Color`.

---

## 5. Standard Library Examples

The standard library uses supertraits extensively. Here are some of the most important ones:

### `Error: Display + Debug`

```rust
// In std::error:
pub trait Error: fmt::Display + fmt::Debug {
    fn source(&self) -> Option<&(dyn Error + 'static)> { None }
    // ...
}
```

Every error type must be both displayable (for user-facing messages) and debuggable
(for developer diagnostics). This is why `#[derive(Debug)]` is almost always needed
on error types.

### `Ord: Eq + PartialOrd<Self>`

```rust
// In std::cmp:
pub trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;
    // ...
}
```

Total ordering (`Ord`) requires:
- `Eq` — reflexive equality (`a == a` is always true)
- `PartialOrd` — partial comparison (some pairs are comparable)

This hierarchy ensures mathematical consistency.

### `Copy: Clone`

```rust
// In std::marker:
pub trait Copy: Clone { }
```

`Copy` is a supertrait of `Clone`. Every `Copy` type can also be `Clone`d.
This makes sense because implicit bitwise copy is a stricter form of cloning.

### `DerefMut: Deref`

```rust
// In std::ops:
pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

Mutable dereferencing requires immutable dereferencing as a baseline.

### `DoubleEndedIterator: Iterator`

```rust
// In std::iter:
pub trait DoubleEndedIterator: Iterator {
    fn next_back(&mut self) -> Option<Self::Item>;
    // ...
}
```

You can iterate from the back only if you can iterate at all.

### Quick Reference Table

| Subtrait | Supertrait(s) | Why |
|----------|---------------|-----|
| `Error` | `Display + Debug` | Errors must be printable and debuggable |
| `Ord` | `Eq + PartialOrd` | Total order requires equality and partial order |
| `Copy` | `Clone` | Bitwise copy is a special case of cloning |
| `DerefMut` | `Deref` | Mutable deref extends immutable deref |
| `DoubleEndedIterator` | `Iterator` | Back-iteration extends forward-iteration |
| `ExactSizeIterator` | `Iterator` | Knowing the size requires being an iterator |
| `FnMut` | `FnOnce` | Callable-by-mut-ref implies callable-by-value |
| `Fn` | `FnMut` | Callable-by-ref implies callable-by-mut-ref |

---

## 6. Supertraits vs OOP Inheritance

If you come from Java, C++, or C#, supertraits might look like class inheritance.
They are **not the same thing**.

### What Supertraits Do

- Require method implementations (behavioral contract)
- Allow calling supertrait methods in default method bodies
- Create a **"requires"** relationship between traits

### What Supertraits Do NOT Do

- **No data inheritance** — supertraits don't give you fields
- **No method override** — you can't override a supertrait's method through the subtrait
- **No vtable hierarchy** — each trait has its own vtable (for trait objects)
- **No "is-a" relationship** on types — a struct doesn't "become" another struct

### Side-by-Side Comparison

```text
OOP Inheritance (Java)              Rust Supertraits
─────────────────────               ────────────────
class Animal {                      trait Animal {
    String name;           ←──×     // No fields!
    void speak() { }                fn speak(&self);
}                                   }

class Dog extends Animal {          trait Dog: Animal {
    // inherits name       ←──×     // No field inheritance
    // inherits speak()             // Must still impl Animal
    void fetch() { }                fn fetch(&self);
}                                   }
```

In Rust, `Dog: Animal` means "anything that is a `Dog` must also be an `Animal`."
It does **not** mean that `Dog` inherits data or code from `Animal`.

### The Rust Way: Composition Over Inheritance

```rust
trait HasName {
    fn name(&self) -> &str;
}

trait CanSpeak: HasName {
    fn speak(&self) {
        println!("{} speaks!", self.name());
    }
}

struct Dog {
    name: String,   // Data is in the struct, not in the trait
}

impl HasName for Dog {
    fn name(&self) -> &str {
        &self.name
    }
}

impl CanSpeak for Dog {} // Uses the default speak() method

fn main() {
    let d = Dog { name: "Rex".to_string() };
    d.speak(); // "Rex speaks!"
}
```

---

## 7. Building Trait Hierarchies

You can design layered abstractions using supertraits, much like building blocks.

### Example: Geometry Hierarchy

```rust
use std::fmt;

// Level 0: Everything must be displayable and named
trait Shape: fmt::Display {
    fn name(&self) -> &str;
}

// Level 1: 2D shapes have area
trait Area: Shape {
    fn area(&self) -> f64;
}

// Level 2: 3D shapes have volume (and also area for their surface)
trait Volume: Area {
    fn volume(&self) -> f64;
}
```

This creates a chain: `Volume` → `Area` → `Shape` → `Display`.

Any type implementing `Volume` must implement **all four traits**.

### Implementing the Hierarchy

```rust
struct Circle {
    radius: f64,
}

impl fmt::Display for Circle {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Circle(r={})", self.radius)
    }
}

impl Shape for Circle {
    fn name(&self) -> &str {
        "Circle"
    }
}

impl Area for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}
```

```rust
struct Sphere {
    radius: f64,
}

impl fmt::Display for Sphere {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Sphere(r={})", self.radius)
    }
}

impl Shape for Sphere {
    fn name(&self) -> &str {
        "Sphere"
    }
}

impl Area for Sphere {
    fn area(&self) -> f64 {
        4.0 * std::f64::consts::PI * self.radius * self.radius
    }
}

impl Volume for Sphere {
    fn volume(&self) -> f64 {
        (4.0 / 3.0) * std::f64::consts::PI * self.radius.powi(3)
    }
}
```

### Using the Hierarchy in Functions

```rust
fn print_area(shape: &dyn Area) {
    // We can call Display methods because Area: Shape: Display
    println!("{} has area {:.2}", shape, shape.area());
}

fn print_volume(solid: &dyn Volume) {
    // We can call Area *and* Display methods
    println!("{} has area {:.2} and volume {:.2}",
        solid, solid.area(), solid.volume());
}

fn main() {
    let c = Circle { radius: 5.0 };
    let s = Sphere { radius: 3.0 };

    print_area(&c);    // Circle(r=5) has area 78.54
    print_area(&s);    // Sphere(r=3) has area 113.10
    print_volume(&s);  // Sphere(r=3) has area 113.10 and volume 113.10
}
```

### Diamond Pattern

What if two supertraits share a common supertrait?

```rust
trait Base {
    fn base_method(&self);
}

trait Left: Base {
    fn left_method(&self);
}

trait Right: Base {
    fn right_method(&self);
}

trait Diamond: Left + Right {
    fn diamond_method(&self);
}
```

```text
        Base
       /    \
     Left   Right
       \    /
       Diamond
```

In Rust, this is **not a problem**. There's only one implementation of `Base` for
any concrete type. No ambiguity, no duplication, no "deadly diamond of death."

```rust
struct Widget;

impl Base for Widget {
    fn base_method(&self) { println!("base"); }
}
impl Left for Widget {
    fn left_method(&self) { println!("left"); }
}
impl Right for Widget {
    fn right_method(&self) { println!("right"); }
}
impl Diamond for Widget {
    fn diamond_method(&self) { println!("diamond"); }
}
```

---

## 8. Supertraits in Other Languages

The concept of one interface or trait requiring another exists across many languages,
but the mechanics differ.

### Java — `extends` for Interfaces

```java
interface Display {
    String display();
}

interface Printable extends Display {
    default void print() {
        System.out.println(display());
    }
}
```

Java interfaces can extend multiple interfaces with `extends`, similar to Rust supertraits.

### C++ — Virtual Inheritance

```cpp
class Display {
public:
    virtual std::string display() const = 0;
};

class Printable : public virtual Display {
public:
    void print() const {
        std::cout << display() << std::endl;
    }
};
```

C++ uses virtual inheritance to handle the diamond problem. Rust avoids this
complexity entirely since traits carry no data.

### Haskell — Class Constraints

```haskell
class (Show a) => Printable a where
    printVal :: a -> IO ()
    printVal x = putStrLn (show x)
```

Haskell's typeclass constraints (`Show a =>`) are the closest analogue to Rust's
supertraits. Both are compile-time requirements on type parameters.

### Swift — Protocol Inheritance

```swift
protocol Display {
    func display() -> String
}

protocol Printable: Display {
    func printValue()
}

extension Printable {
    func printValue() {
        print(display())
    }
}
```

Swift protocols can inherit from other protocols, very similar to Rust supertraits.

### Go — Embedding

```go
type Display interface {
    Display() string
}

type Printable interface {
    Display  // embedded
    Print()
}
```

Go uses interface embedding rather than explicit inheritance syntax. The effect is
similar: `Printable` requires all methods of `Display`.

### Comparison Table

| Language | Mechanism | Syntax | Diamond Problem | Data Inherited? |
|----------|-----------|--------|-----------------|-----------------|
| **Rust** | Supertraits | `trait A: B` | No issue | No |
| **Java** | Interface `extends` | `interface A extends B` | No issue (interfaces) | No |
| **C++** | Virtual inheritance | `class A : virtual B` | Solved via `virtual` | Yes (problematic) |
| **Haskell** | Class constraints | `class (B a) => A a` | No issue | No (no data) |
| **Swift** | Protocol inheritance | `protocol A: B` | No issue | No |
| **Go** | Embedding | embedded interface | No issue | No |

---

## 9. Associated Types with Supertraits

You can access a supertrait's associated types in your subtrait.

### Basic Example

```rust
trait Container {
    type Item;
    fn items(&self) -> &[Self::Item];
}

trait SortableContainer: Container
where
    Self::Item: Ord,
{
    fn sorted_items(&self) -> Vec<&Self::Item> {
        let mut refs: Vec<&Self::Item> = self.items().iter().collect();
        refs.sort();
        refs
    }
}
```

Here `SortableContainer` requires `Container` and further constrains `Container::Item`
to be `Ord`.

### Chaining Associated Types

```rust
trait Graph {
    type Node;
    type Edge;

    fn nodes(&self) -> Vec<&Self::Node>;
    fn edges(&self) -> Vec<&Self::Edge>;
}

trait WeightedGraph: Graph {
    type Weight;

    fn edge_weight(&self, edge: &Self::Edge) -> Self::Weight;
}

trait ShortestPath: WeightedGraph
where
    Self::Weight: std::ops::Add<Output = Self::Weight> + Ord + Default + Copy,
{
    fn shortest_path(&self, from: &Self::Node, to: &Self::Node) -> Option<Self::Weight>;
}
```

Each level of the hierarchy adds more associated types and constraints,
building up a rich abstraction.

### Using Associated Types in Bounds

```rust
use std::fmt;

trait Describable {
    type Description: fmt::Display;
    fn describe(&self) -> Self::Description;
}

trait Loggable: Describable {
    fn log(&self) {
        // We can use {} because Description: Display
        println!("[LOG] {}", self.describe());
    }
}

struct Event {
    message: String,
}

impl Describable for Event {
    type Description = String;
    fn describe(&self) -> String {
        self.message.clone()
    }
}

impl Loggable for Event {}

fn main() {
    let e = Event { message: "User logged in".to_string() };
    e.log(); // [LOG] User logged in
}
```

---

## 10. Exercises

### Exercise 1: Summarizable Report

**Goal:** Create a trait hierarchy for summarizable, printable reports.

1. Define a trait `Summarizable` with a method `fn summary(&self) -> String`.
2. Define a trait `Report: Summarizable + fmt::Display` with a default method
   `fn print_report(&self)` that prints the display output followed by the summary.
3. Implement both traits for a `SalesReport` struct with fields `region: String`
   and `total: f64`.

<details>
<summary><strong>Solution</strong></summary>

```rust
use std::fmt;

trait Summarizable {
    fn summary(&self) -> String;
}

trait Report: Summarizable + fmt::Display {
    fn print_report(&self) {
        println!("=== Report ===");
        println!("{}", self);
        println!("Summary: {}", self.summary());
        println!("==============");
    }
}

struct SalesReport {
    region: String,
    total: f64,
}

impl fmt::Display for SalesReport {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Sales Report for {}: ${:.2}", self.region, self.total)
    }
}

impl Summarizable for SalesReport {
    fn summary(&self) -> String {
        if self.total > 10_000.0 {
            format!("{} exceeded target!", self.region)
        } else {
            format!("{} below target.", self.region)
        }
    }
}

impl Report for SalesReport {}

fn main() {
    let report = SalesReport {
        region: "West Coast".to_string(),
        total: 15_230.50,
    };
    report.print_report();
}
```

**Output:**
```text
=== Report ===
Sales Report for West Coast: $15230.50
Summary: West Coast exceeded target!
==============
```

</details>

---

### Exercise 2: Layered Animal Hierarchy

**Goal:** Build a three-level trait hierarchy for animals.

1. `trait Named` — requires `fn name(&self) -> &str`
2. `trait Animal: Named + fmt::Display` — requires `fn species(&self) -> &str`
   and has a default method `fn introduce(&self)` that prints *"Hi, I'm [name] the [species]"*
3. `trait Trainable: Animal` — requires `fn tricks(&self) -> Vec<String>` and has a
   default method `fn show_tricks(&self)` that prints each trick
4. Implement all three for a `Dog` struct

<details>
<summary><strong>Solution</strong></summary>

```rust
use std::fmt;

trait Named {
    fn name(&self) -> &str;
}

trait Animal: Named + fmt::Display {
    fn species(&self) -> &str;

    fn introduce(&self) {
        println!("Hi, I'm {} the {}", self.name(), self.species());
    }
}

trait Trainable: Animal {
    fn tricks(&self) -> Vec<String>;

    fn show_tricks(&self) {
        println!("{} can do:", self.name());
        for trick in self.tricks() {
            println!("  - {}", trick);
        }
    }
}

struct Dog {
    name: String,
    tricks: Vec<String>,
}

impl fmt::Display for Dog {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "🐕 {} (knows {} tricks)", self.name, self.tricks.len())
    }
}

impl Named for Dog {
    fn name(&self) -> &str {
        &self.name
    }
}

impl Animal for Dog {
    fn species(&self) -> &str {
        "Dog"
    }
}

impl Trainable for Dog {
    fn tricks(&self) -> Vec<String> {
        self.tricks.clone()
    }
}

fn main() {
    let dog = Dog {
        name: "Buddy".to_string(),
        tricks: vec!["sit".to_string(), "shake".to_string(), "roll over".to_string()],
    };

    dog.introduce();    // Hi, I'm Buddy the Dog
    dog.show_tricks();  // Buddy can do:  - sit  - shake  - roll over
    println!("{}", dog); // 🐕 Buddy (knows 3 tricks)
}
```

</details>

---

### Exercise 3: Custom Error Type with Supertraits

**Goal:** Create a custom error type that satisfies the `std::error::Error` supertrait
requirements (`Display + Debug`).

1. Define an enum `AppError` with variants `NotFound(String)` and `PermissionDenied(String)`
2. Implement `Debug`, `Display`, and `Error` for it
3. Write a function `fn open_file(name: &str) -> Result<String, AppError>` that
   returns `NotFound` if the name is `"missing.txt"`, otherwise returns `Ok`
4. Demonstrate using `?` in `main()`

<details>
<summary><strong>Solution</strong></summary>

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
enum AppError {
    NotFound(String),
    PermissionDenied(String),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(name) => write!(f, "File not found: {}", name),
            AppError::PermissionDenied(name) => write!(f, "Permission denied: {}", name),
        }
    }
}

// Error requires Display + Debug — both are satisfied
impl Error for AppError {}

fn open_file(name: &str) -> Result<String, AppError> {
    match name {
        "missing.txt" => Err(AppError::NotFound(name.to_string())),
        "secret.txt" => Err(AppError::PermissionDenied(name.to_string())),
        _ => Ok(format!("Contents of {}", name)),
    }
}

fn main() -> Result<(), Box<dyn Error>> {
    let contents = open_file("hello.txt")?;
    println!("{}", contents);

    let result = open_file("missing.txt");
    match result {
        Ok(c) => println!("{}", c),
        Err(e) => {
            println!("Error: {}", e);          // Display
            println!("Debug: {:?}", e);        // Debug
            println!("Source: {:?}", e.source()); // Error trait method
        }
    }

    Ok(())
}
```

**Output:**
```text
Contents of hello.txt
Error: File not found: missing.txt
Debug: NotFound("missing.txt")
Source: None
```

</details>

---

## 11. Summary

### Key Concepts at a Glance

| Concept | Description | Example |
|---------|-------------|---------|
| **Supertrait** | A trait required by another trait | `Display` in `trait Printable: Display` |
| **Subtrait** | The trait that declares the requirement | `Printable` in `trait Printable: Display` |
| **Multiple supertraits** | Requiring more than one trait | `trait Error: Display + Debug` |
| **Transitive requirement** | If `C: B` and `B: A`, then `C` requires `A` | `Volume: Area: Shape` |
| **Diamond pattern** | Two supertraits sharing a common supertrait | Safe in Rust — one impl per type |
| **No data inheritance** | Supertraits only require method implementations | Unlike OOP class inheritance |
| **Associated types** | Subtrait can access supertrait's associated types | `Self::Item` from `Container` |

### When to Use Supertraits

- ✅ Your default methods need to call methods from another trait
- ✅ There's a logical "is-a" relationship between capabilities
- ✅ You want the compiler to enforce that prerequisites are met
- ✅ You're building layered abstractions (Shape → Area → Volume)
- ❌ Don't use them just to bundle unrelated traits — use `+` bounds on functions instead

### The Golden Rule

> Supertraits are **requirements**, not **inheritance**. They say *"you must also implement X"*,
> not *"you get X's code and data for free."*

---

[← Previous: Operator Overloading](07-operator-overloading.md) | [Next: Static vs Dynamic Dispatch →](09-static-vs-dynamic-dispatch.md)

---

*Tutorial 8 of 9 — Stage 8: Generics & Traits*

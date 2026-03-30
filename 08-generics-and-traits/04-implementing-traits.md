# Implementing Traits 🎯

> **"A trait is just a promise — `impl Trait for Type` is where you keep it."**

---

## Table of Contents

- [Basic Trait Implementation](#basic-trait-implementation)
- [Implementing Multiple Traits](#implementing-multiple-traits)
- [The Orphan Rule](#the-orphan-rule)
- [The Newtype Pattern](#the-newtype-pattern)
- [Blanket Implementations](#blanket-implementations)
- [Derive Macros — Auto-Implementing Traits](#derive-macros--auto-implementing-traits)
- [Trait Coherence — No Conflicting Implementations](#trait-coherence--no-conflicting-implementations)
- [Implementing Traits for Generic Types](#implementing-traits-for-generic-types)
- [Drop and Destructor Trait](#drop-and-destructor-trait)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Basic Trait Implementation

In the previous tutorial we learned how to _define_ a trait — a contract that a type
can promise to fulfil.  Now we fulfil that promise with the `impl Trait for Type`
syntax.

### The Full Picture: Struct → Trait → Impl

```rust
// 1. Define the data
struct NewsArticle {
    title: String,
    author: String,
    content: String,
}

// 2. Define the contract
trait Summary {
    fn summarize(&self) -> String;
}

// 3. Fulfil the contract
impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{} by {} — {}", self.title, self.author, &self.content[..50])
    }
}
```

Let's use it:

```rust
fn main() {
    let article = NewsArticle {
        title: String::from("Rust 2026 Edition Released"),
        author: String::from("The Rust Team"),
        content: String::from(
            "The Rust 2026 edition introduces several exciting new features \
             that continue to improve developer ergonomics..."
        ),
    };

    println!("{}", article.summarize());
    // Rust 2026 Edition Released by The Rust Team — The Rust 2026 edition introduces several exci
}
```

The three pieces — struct, trait, and impl block — can live in different modules or
even different crates (with restrictions we'll discuss shortly).

### Implementing a Trait with Default Methods

Recall that traits can provide default implementations.  When you `impl` the trait you
can choose to _override_ them or let the defaults stand:

```rust
trait Summary {
    fn summarize_author(&self) -> String;

    // Default implementation that calls the required method
    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}

struct Tweet {
    username: String,
    body: String,
}

impl Summary for Tweet {
    // Only the required method — we accept the default `summarize`
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}

fn main() {
    let tweet = Tweet {
        username: String::from("rustlang"),
        body: String::from("Exciting news!"),
    };

    // Uses the default summarize, which calls our summarize_author
    println!("{}", tweet.summarize());
    // (Read more from @rustlang...)
}
```

> **Key insight:** A default method can call other methods in the same trait — even
> ones that have no default.  The concrete impl supplies the "abstract" pieces, and
> the defaults wire everything together.

---

## Implementing Multiple Traits

A type can implement **as many traits as you like**.  There is no upper limit:

```rust
use std::fmt;

struct Point {
    x: f64,
    y: f64,
}

// Trait 1 — Display (from std)
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

// Trait 2 — Debug (manually, instead of derive)
impl fmt::Debug for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Point {{ x: {}, y: {} }}", self.x, self.y)
    }
}

// Trait 3 — Clone
impl Clone for Point {
    fn clone(&self) -> Self {
        Point { x: self.x, y: self.y }
    }
}

// Trait 4 — Our custom trait
trait Summary {
    fn summarize(&self) -> String;
}

impl Summary for Point {
    fn summarize(&self) -> String {
        format!("Point at {}", self)   // uses our Display impl
    }
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };

    println!("Display:  {}", p);           // (3, 4)
    println!("Debug:    {:?}", p);         // Point { x: 3, y: 4 }
    println!("Summary:  {}", p.summarize()); // Point at (3, 4)

    let p2 = p.clone();
    println!("Cloned:   {}", p2);          // (3, 4)
}
```

Each `impl` block is independent. The compiler checks them all, and the type ends
up satisfying every trait it implements.

### When Methods Collide

If two traits define a method with the same name, Rust won't guess which one you
mean — you must use **fully qualified syntax**:

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) { println!("This is your captain speaking."); }
}

impl Wizard for Human {
    fn fly(&self) { println!("*levitates*"); }
}

impl Human {
    fn fly(&self) { println!("*waves arms furiously*"); }
}

fn main() {
    let h = Human;

    h.fly();               // calls Human::fly — the inherent method
    Pilot::fly(&h);        // calls <Human as Pilot>::fly
    Wizard::fly(&h);       // calls <Human as Wizard>::fly
}
```

The fully qualified form `<Type as Trait>::method(...)` resolves any ambiguity.

---

## The Orphan Rule

This is one of the most important rules in Rust's type system.  It often surprises
newcomers, so let's spell it out clearly.

### The Rule

> You can implement a trait for a type **only if at least one of the following is
> defined in your crate**:
>
> - The **trait**, or
> - The **type**

This is called the **orphan rule** because without it an `impl` would be an "orphan"
— belonging to neither the crate that defined the trait nor the crate that defined
the type.

### What You CAN Do

| Trait defined in… | Type defined in… | Allowed? |
|---|---|---|
| Your crate | Your crate | ✅ Yes |
| Your crate | std / external | ✅ Yes |
| std / external | Your crate | ✅ Yes |
| std / external | std / external | ❌ **No** |

```rust
// ✅ Your trait, foreign type
trait Printable {
    fn print(&self);
}

impl Printable for Vec<String> {
    fn print(&self) {
        for s in self {
            println!("{}", s);
        }
    }
}

// ✅ Foreign trait, your type
use std::fmt;

struct Color(u8, u8, u8);

impl fmt::Display for Color {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "#{:02X}{:02X}{:02X}", self.0, self.1, self.2)
    }
}
```

```rust
// ❌ Foreign trait, foreign type — COMPILE ERROR
impl fmt::Display for Vec<String> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // ...
    }
}
// error[E0117]: only traits defined in the current crate can be implemented
//               for types defined outside of the crate
```

### Why Does the Orphan Rule Exist?

Imagine two crates, `crate_a` and `crate_b`, both decide to implement
`Display for Vec<String>`.  A third crate depends on both.  Which implementation
wins?  There's no sane answer.

The orphan rule **prevents this entirely** — at most one crate can ever provide a
given `(trait, type)` pair, so the compiler never encounters conflicting
implementations.

This is also a foundation of **trait coherence**, which we explore in more depth
[below](#trait-coherence--no-conflicting-implementations).

---

## The Newtype Pattern

The orphan rule is strict — so what do you do when you _really_ need to implement
a foreign trait for a foreign type?

The answer: **wrap the type**.

### The Pattern

```rust
use std::fmt;

// A thin wrapper — a tuple struct with one field
struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // self.0 accesses the inner Vec<String>
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![
        String::from("hello"),
        String::from("world"),
    ]);

    println!("{}", w);  // [hello, world]
}
```

`Wrapper` is your type, so implementing any trait for it is fair game.

### Making the Wrapper Transparent

The downside: `Wrapper` doesn't have `Vec`'s methods.  You can solve this several
ways:

**1. Implement `Deref`** to automatically delegate method calls:

```rust
use std::ops::Deref;

impl Deref for Wrapper {
    type Target = Vec<String>;

    fn deref(&self) -> &Vec<String> {
        &self.0
    }
}

fn main() {
    let w = Wrapper(vec![String::from("a"), String::from("b")]);

    // Now Vec methods work through Deref:
    println!("Length: {}", w.len());       // 2
    println!("First: {}", &w[0]);          // a
    println!("Display: {}", w);            // [a, b]
}
```

**2. Add an `into_inner` method** to unwrap when you're done:

```rust
impl Wrapper {
    fn into_inner(self) -> Vec<String> {
        self.0
    }
}
```

> **When to reach for the newtype pattern:**
> - You want to implement a foreign trait for a foreign type
> - You want to give a type a more descriptive name (e.g., `Meters(f64)` vs plain `f64`)
> - You want to enforce invariants that the inner type doesn't guarantee
> - You want to hide the inner type's API and expose only a subset

---

## Blanket Implementations

A **blanket implementation** implements a trait for _every_ type that satisfies
some bound.  This is one of Rust's most powerful features.

### How `to_string()` Works

Ever wonder why you can call `.to_string()` on anything that implements `Display`?

```rust
// In the standard library (simplified):
impl<T: fmt::Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{}", self)
    }
}
```

This single `impl` block covers **every type in existence** (past, present, and
future) that implements `Display`.  That's a blanket implementation.

```rust
fn main() {
    let n: i32 = 42;
    let s: String = n.to_string();   // works because i32: Display

    let f: f64 = 3.14;
    let s2: String = f.to_string();  // works because f64: Display

    // Your own types too:
    struct Celsius(f64);

    impl std::fmt::Display for Celsius {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "{}°C", self.0)
        }
    }

    let temp = Celsius(100.0);
    let s3 = temp.to_string();       // works! Celsius: Display → Celsius: ToString
    println!("{}", s3);              // 100°C
}
```

### Writing Your Own Blanket Impl

```rust
trait Describe {
    fn describe(&self) -> String;
}

// Every type that implements Display + Debug gets Describe for free
impl<T: std::fmt::Display + std::fmt::Debug> Describe for T {
    fn describe(&self) -> String {
        format!("Display: {} | Debug: {:?}", self, self)
    }
}

fn main() {
    println!("{}", 42.describe());
    // Display: 42 | Debug: 42

    println!("{}", "hello".describe());
    // Display: hello | Debug: "hello"

    println!("{}", vec![1, 2, 3].describe());
    // Display won't work for Vec — Vec doesn't implement Display!
    // This line won't compile.
}
```

> **Blanket impls and the orphan rule:** Because a blanket impl potentially covers
> types from other crates, only the crate that defines the trait can write a blanket
> impl.  This prevents conflicts.

### Common Blanket Impls in `std`

| Blanket impl | Effect |
|---|---|
| `impl<T: Display> ToString for T` | Any `Display` type gets `.to_string()` |
| `impl<T> From<T> for T` | Every type can convert from itself |
| `impl<T: ?Sized> Borrow<T> for T` | Every type can borrow as itself |
| `impl<T> AsRef<T> for T` | Every type can ref-convert to itself |

---

## Derive Macros — Auto-Implementing Traits

Writing `impl` blocks by hand is educational, but often tedious. Rust's **derive
macros** generate trait implementations automatically based on the struct's fields.

### Basic Usage

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = p1.clone();

    println!("{:?}", p1);           // Point { x: 1.0, y: 2.0 }
    println!("Equal? {}", p1 == p2); // Equal? true
}
```

One line — `#[derive(Debug, Clone, PartialEq)]` — replaced three entire `impl`
blocks.

### Derivable Standard-Library Traits

| Trait | What It Does | Requirements |
|---|---|---|
| `Debug` | `{:?}` formatting | All fields implement `Debug` |
| `Clone` | `.clone()` deep copy | All fields implement `Clone` |
| `Copy` | Implicit bitwise copy | All fields implement `Copy` (+ `Clone`) |
| `PartialEq` | `==` and `!=` | All fields implement `PartialEq` |
| `Eq` | Marker: equality is total | All fields implement `Eq` (+ `PartialEq`) |
| `PartialOrd` | `<`, `>`, `<=`, `>=` | All fields implement `PartialOrd` |
| `Ord` | Total ordering | All fields implement `Ord` (+ `PartialOrd` + `Eq`) |
| `Hash` | Hashing for `HashMap` keys | All fields implement `Hash` |
| `Default` | `Type::default()` | All fields implement `Default` |

### How Derive Works Under the Hood

The compiler inspects each field and generates the obvious recursive implementation.
For example, `#[derive(PartialEq)]` on this struct:

```rust
#[derive(PartialEq)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
}
```

...generates roughly:

```rust
impl PartialEq for Color {
    fn eq(&self, other: &Self) -> bool {
        self.r == other.r && self.g == other.g && self.b == other.b
    }
}
```

Each field is compared using its own `PartialEq` implementation. If any field
doesn't implement the trait, the derive fails at compile time.

### When Derive Doesn't Work

**1. A field doesn't implement the trait:**

```rust
struct Handle {
    raw: *mut u8,    // raw pointers don't implement Clone
}

// #[derive(Clone)] on a struct containing Handle won't compile
```

**2. You need custom logic:**

```rust
struct CaseInsensitiveString(String);

// derive(PartialEq) would compare bytes — we want case-insensitive comparison
impl PartialEq for CaseInsensitiveString {
    fn eq(&self, other: &Self) -> bool {
        self.0.to_lowercase() == other.0.to_lowercase()
    }
}
```

**3. Performance considerations:**

```rust
// derive(Clone) for a large struct clones every field
// Maybe you want a cheaper clone that shares some data via Rc/Arc
```

### Third-Party Derives: Serde

The derive system is extensible through **procedural macros**.  The most famous
third-party example is `serde`:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    name: String,
    version: u32,
    features: Vec<String>,
}

fn main() {
    let config = Config {
        name: String::from("my-app"),
        version: 1,
        features: vec![String::from("logging"), String::from("metrics")],
    };

    // Serialize to JSON
    let json = serde_json::to_string_pretty(&config).unwrap();
    println!("{}", json);

    // Deserialize back
    let parsed: Config = serde_json::from_str(&json).unwrap();
    println!("{:?}", parsed);
}
```

Other popular third-party derives: `thiserror::Error`, `clap::Parser`,
`sqlx::FromRow`, `strum::Display`.

---

## Trait Coherence — No Conflicting Implementations

**Coherence** is the property that for any given `(trait, type)` pair, there is
_at most one_ implementation.  The orphan rule is a _mechanism_ that enforces
coherence, but the _goal_ is coherence itself.

### The Core Guarantee

```text
For any type T and trait Foo:
  ∃ at most ONE impl Foo for T
  across ALL crates in the dependency graph.
```

This means the compiler can always unambiguously resolve which code to run when
you call a trait method.

### Why Coherence Matters

Without coherence, you get the kind of subtle, catastrophic bugs that plague C++:

```text
// Hypothetical: NO coherence

// crate_a
impl Display for Vec<i32> {
    fn fmt(...) { /* format as JSON array */ }
}

// crate_b
impl Display for Vec<i32> {
    fn fmt(...) { /* format as comma-separated */ }
}

// crate_c (depends on both)
fn print_vec(v: &Vec<i32>) {
    println!("{}", v);  // Which impl? Depends on link order? Undefined behavior?
}
```

Rust says: **no**.  This is a compile-time error.  The orphan rule ensures that
`crate_a` and `crate_b` can't both write `impl Display for Vec<i32>` because
neither crate owns `Display` nor `Vec<i32>`.

### Coherence Across Languages

| Language | Coherence? | Notes |
|---|---|---|
| **Rust** | ✅ Strict | Orphan rule enforced at compile time |
| **Haskell** | ⚠️ Weak | Orphan instances allowed but discouraged; can cause subtle bugs |
| **C++** | ❌ None | ODR violations are undefined behavior; no concept checking (until C++20 concepts, which still don't enforce coherence) |
| **Java** | N/A | Single dispatch via class hierarchy, no external implementation of interfaces for existing classes |
| **Swift** | ⚠️ Partial | Protocol conformances can conflict across modules; runtime behavior may vary |

Rust's strict approach trades some flexibility (you can't always impl what you
want) for **rock-solid guarantees** — if it compiles, there's exactly one
implementation for every trait-type pair.

### Negative Reasoning

Coherence lets the compiler do **negative reasoning**: it can conclude that a type
does _not_ implement a trait and use that information during type-checking.  For
example, the compiler knows `u8` does not implement `Iterator`, and it doesn't
have to worry about someone adding that impl in another crate — the orphan rule
prevents it.

---

## Implementing Traits for Generic Types

You can implement a trait for a **generic** type, optionally with bounds on the
type parameter.

### Basic Generic Impl

```rust
trait Summary {
    fn summarize(&self) -> String;
}

// Implement Summary for Vec<T> where T: std::fmt::Display
impl<T: std::fmt::Display> Summary for Vec<T> {
    fn summarize(&self) -> String {
        let items: Vec<String> = self.iter().map(|x| x.to_string()).collect();
        format!("[{}] ({} items)", items.join(", "), self.len())
    }
}

fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    println!("{}", nums.summarize());
    // [1, 2, 3, 4, 5] (5 items)

    let names = vec!["Alice", "Bob"];
    println!("{}", names.summarize());
    // [Alice, Bob] (2 items)
}
```

The `impl<T: Display>` means "for _any_ T that implements Display, implement
Summary for `Vec<T>`." The compiler will generate specialized versions for each
concrete `T` used.

### Conditional Method Implementation

You can also add methods to a generic struct only when certain bounds are met:

```rust
struct Pair<T> {
    first: T,
    second: T,
}

// These methods exist for ALL Pair<T>
impl<T> Pair<T> {
    fn new(first: T, second: T) -> Self {
        Pair { first, second }
    }
}

// These methods ONLY exist when T: PartialOrd + Display
impl<T: std::fmt::Display + PartialOrd> Pair<T> {
    fn larger(&self) -> &T {
        if self.first >= self.second {
            &self.first
        } else {
            &self.second
        }
    }

    fn print_larger(&self) {
        println!("The larger value is: {}", self.larger());
    }
}

fn main() {
    let pair = Pair::new(5, 10);
    pair.print_larger();  // The larger value is: 10

    // Pair<SomeNonDisplayType> wouldn't have print_larger at all
}
```

This pattern is extremely common in the standard library — `Vec<T>` has different
methods available depending on whether `T` is `Clone`, `Ord`, `PartialEq`, etc.

---

## Drop and Destructor Trait

The `Drop` trait is Rust's **destructor** — it lets you run custom cleanup code when
a value goes out of scope.

### The Trait

```rust
trait Drop {
    fn drop(&mut self);
}
```

That's it — one method.  The compiler calls it automatically; you never call
`.drop()` directly (it's a compile error to try).

### Implementing Drop

```rust
struct DatabaseConnection {
    url: String,
    id: u32,
}

impl DatabaseConnection {
    fn new(url: &str, id: u32) -> Self {
        println!("[{}] Connection opened to {}", id, url);
        DatabaseConnection {
            url: url.to_string(),
            id,
        }
    }
}

impl Drop for DatabaseConnection {
    fn drop(&mut self) {
        println!("[{}] Connection to {} closed — resources freed", self.id, self.url);
    }
}

fn main() {
    let conn1 = DatabaseConnection::new("postgres://localhost/db", 1);
    let conn2 = DatabaseConnection::new("postgres://localhost/db", 2);

    println!("Doing work...");

    // conn2 is dropped first (reverse declaration order)
    // conn1 is dropped second
}
```

Output:

```text
[1] Connection opened to postgres://localhost/db
[2] Connection opened to postgres://localhost/db
Doing work...
[2] Connection to postgres://localhost/db closed — resources freed
[1] Connection to postgres://localhost/db closed — resources freed
```

### Connection to RAII (Stage 3 — Ownership)

In Stage 3 we learned about **RAII** — Resource Acquisition Is Initialization.
The `Drop` trait is the _implementation mechanism_ for RAII in Rust:

| RAII concept | Rust mechanism |
|---|---|
| Acquire resource on creation | `new()` / constructor function |
| Release resource on destruction | `impl Drop` |
| Deterministic destruction | Scope-based — no GC, no `finally` |
| Exception safety | `Drop` runs even during stack unwinding (panic) |

### Drop Order Rules

The order in which values are dropped is **deterministic** and follows two rules:

**1. Local variables:** dropped in **reverse order of declaration**

```rust
fn main() {
    let a = Droppable("a");   // declared first
    let b = Droppable("b");   // declared second
    let c = Droppable("c");   // declared third
    // drop order: c, b, a
}
```

**2. Struct fields:** dropped in **declaration order** (NOT reverse)

```rust
struct Combo {
    first: Droppable,    // dropped first
    second: Droppable,   // dropped second
    third: Droppable,    // dropped third
}
```

This distinction matters when fields reference each other or hold resources that
depend on other resources.

### Early Drop with `std::mem::drop`

You can't call `.drop()` directly, but you can use the free function `drop()`:

```rust
fn main() {
    let conn = DatabaseConnection::new("postgres://localhost/db", 1);
    println!("Using connection...");

    drop(conn);  // explicitly drop early
    println!("Connection already freed — doing other work...");
}
```

`std::mem::drop` is literally defined as:

```rust
pub fn drop<T>(_x: T) {}
```

It takes ownership of the value, then does nothing — when the parameter goes out
of scope (immediately), the compiler inserts the `Drop::drop` call.  Beautifully
simple.

### Drop and Copy Are Mutually Exclusive

A type cannot implement both `Drop` and `Copy`.  Why?  `Copy` means implicit
bitwise duplication — if the type also has `Drop`, the compiler wouldn't know how
many times to run the destructor.  So Rust forbids it:

```rust
#[derive(Copy, Clone)]
struct Bad {
    data: i32,
}

impl Drop for Bad {
    fn drop(&mut self) {}  // ERROR: Copy and Drop are mutually exclusive
}
```

---

## Exercises

### Exercise 1: Implementing a Trait for Multiple Types

Define a trait `Area` with a method `fn area(&self) -> f64`.  Implement it for three
types: `Circle`, `Rectangle`, and `Triangle`.  Then write a function
`fn print_area(shape: &impl Area)` that prints the area.

```rust
use std::f64::consts::PI;

trait Area {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

struct Triangle {
    base: f64,
    height: f64,
}

// TODO: Implement Area for Circle (π * r²)
// TODO: Implement Area for Rectangle (width * height)
// TODO: Implement Area for Triangle (0.5 * base * height)

fn print_area(shape: &impl Area) {
    println!("Area: {:.2}", shape.area());
}

fn main() {
    let c = Circle { radius: 5.0 };
    let r = Rectangle { width: 4.0, height: 6.0 };
    let t = Triangle { base: 3.0, height: 8.0 };

    print_area(&c);  // Area: 78.54
    print_area(&r);  // Area: 24.00
    print_area(&t);  // Area: 12.00
}
```

<details>
<summary>Solution</summary>

```rust
use std::f64::consts::PI;

trait Area {
    fn area(&self) -> f64;
}

struct Circle { radius: f64 }
struct Rectangle { width: f64, height: f64 }
struct Triangle { base: f64, height: f64 }

impl Area for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }
}

impl Area for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

impl Area for Triangle {
    fn area(&self) -> f64 {
        0.5 * self.base * self.height
    }
}

fn print_area(shape: &impl Area) {
    println!("Area: {:.2}", shape.area());
}

fn main() {
    let c = Circle { radius: 5.0 };
    let r = Rectangle { width: 4.0, height: 6.0 };
    let t = Triangle { base: 3.0, height: 8.0 };

    print_area(&c);  // Area: 78.54
    print_area(&r);  // Area: 24.00
    print_area(&t);  // Area: 12.00
}
```

</details>

### Exercise 2: Newtype Pattern + Drop

Create a `TempFile` newtype that wraps a `String` (the filename).  Implement
`Drop` so it prints "Cleaning up temp file: {name}" when it goes out of scope.
Then implement `Display` so it prints the filename.  Create three `TempFile`
instances and observe the drop order.

```rust
use std::fmt;

struct TempFile(String);

// TODO: Implement Display for TempFile
// TODO: Implement Drop for TempFile (prints cleanup message)

fn main() {
    let f1 = TempFile(String::from("/tmp/data_001.csv"));
    let f2 = TempFile(String::from("/tmp/data_002.csv"));
    let f3 = TempFile(String::from("/tmp/data_003.csv"));

    println!("Working with: {}, {}, {}", f1, f2, f3);
    println!("Done — dropping...");

    // Expected output after "Done — dropping...":
    //   Cleaning up temp file: /tmp/data_003.csv
    //   Cleaning up temp file: /tmp/data_002.csv
    //   Cleaning up temp file: /tmp/data_001.csv
}
```

<details>
<summary>Solution</summary>

```rust
use std::fmt;

struct TempFile(String);

impl fmt::Display for TempFile {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl Drop for TempFile {
    fn drop(&mut self) {
        println!("Cleaning up temp file: {}", self.0);
    }
}

fn main() {
    let f1 = TempFile(String::from("/tmp/data_001.csv"));
    let f2 = TempFile(String::from("/tmp/data_002.csv"));
    let f3 = TempFile(String::from("/tmp/data_003.csv"));

    println!("Working with: {}, {}, {}", f1, f2, f3);
    println!("Done — dropping...");
}
```

</details>

### Exercise 3: Blanket Implementation

Define a trait `Shout` with one method `fn shout(&self) -> String`.  Write a
**blanket implementation** of `Shout` for every type that implements `Display` —
the `shout` method should return the display string in ALL CAPS with `!!!` appended.
Test it on `&str`, `i32`, and a custom struct.

```rust
use std::fmt;

trait Shout {
    fn shout(&self) -> String;
}

// TODO: Blanket impl — impl<T: ???> Shout for T { ... }

struct Name(String);

// TODO: Implement Display for Name

fn main() {
    println!("{}", "hello world".shout());  // HELLO WORLD!!!
    println!("{}", 42.shout());             // 42!!!
    println!("{}", Name(String::from("rust")).shout());  // RUST!!!
}
```

<details>
<summary>Solution</summary>

```rust
use std::fmt;

trait Shout {
    fn shout(&self) -> String;
}

impl<T: fmt::Display> Shout for T {
    fn shout(&self) -> String {
        format!("{}!!!", self.to_string().to_uppercase())
    }
}

struct Name(String);

impl fmt::Display for Name {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

fn main() {
    println!("{}", "hello world".shout());  // HELLO WORLD!!!
    println!("{}", 42.shout());             // 42!!!
    println!("{}", Name(String::from("rust")).shout());  // RUST!!!
}
```

</details>

---

## Summary

| Concept | Key Idea |
|---|---|
| `impl Trait for Type` | The concrete act of fulfilling a trait's contract |
| Multiple traits | A type can implement any number of traits |
| Orphan rule | Either the trait or the type must be yours |
| Newtype pattern | Wrap a foreign type to bypass the orphan rule |
| Blanket impls | `impl<T: Bound> Trait for T` — implement for all matching types |
| Derive macros | `#[derive(...)]` auto-generates impls from struct fields |
| Coherence | At most one impl per (trait, type) pair, globally |
| Generic impls | `impl<T: Bound> Trait for MyType<T>` — conditional implementations |
| `Drop` trait | Custom destructors — RAII in action |
| Drop + Copy | Mutually exclusive — a type can be one or the other, not both |

**You now know how to _implement_ traits in every way Rust allows.**  In the next
tutorial, we'll learn how to _require_ traits — using **trait bounds** to constrain
generic type parameters, which is where generics and traits truly come together.

---

**Previous:** [← Defining Traits](./03-defining-traits.md) · **Next:** [Trait Bounds →](./05-trait-bounds.md)

<p align="center"><i>Tutorial 4 of 9 — Stage 8: Generics & Traits</i></p>

# Common Standard Library Traits 📚

> **The standard library defines dozens of traits that form the backbone of Rust's ecosystem. Understanding Display, Debug, Clone, Copy, PartialEq, Eq, Hash, Default, From, Into, and others is essential — you'll use them in every Rust program you write.**

---

## Table of Contents

- [The Standard Library Trait Landscape](#the-standard-library-trait-landscape)
- [Display & Debug — Formatting Traits](#display--debug--formatting-traits)
  - [Debug](#debug)
  - [Display](#display)
  - [Other Formatting Traits](#other-formatting-traits)
- [Clone & Copy — Duplication Traits](#clone--copy--duplication-traits)
  - [Clone](#clone)
  - [Copy](#copy)
  - [The Drop-Copy Exclusion](#the-drop-copy-exclusion)
- [PartialEq & Eq — Equality Traits](#partialeq--eq--equality-traits)
  - [PartialEq](#partialeq)
  - [Eq](#eq)
  - [The NaN Problem](#the-nan-problem)
- [PartialOrd & Ord — Ordering Traits](#partialord--ord--ordering-traits)
  - [PartialOrd](#partialord)
  - [Ord — Total Ordering](#ord--total-ordering)
- [Hash — For HashMaps and HashSets](#hash--for-hashmaps-and-hashsets)
- [Default — Sensible Starting Values](#default--sensible-starting-values)
- [From & Into — Infallible Conversions](#from--into--infallible-conversions)
- [TryFrom & TryInto — Fallible Conversions](#tryfrom--tryinto--fallible-conversions)
- [AsRef & AsMut — Cheap Reference Conversions](#asref--asmut--cheap-reference-conversions)
- [Deref & DerefMut — Smart Pointer Magic](#deref--derefmut--smart-pointer-magic)
- [Iterator — The Sequence Trait](#iterator--the-sequence-trait)
- [Drop — The Destructor](#drop--the-destructor)
- [Send & Sync — Thread Safety Markers](#send--sync--thread-safety-markers)
- [The Trait Cheat Sheet](#the-trait-cheat-sheet)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Standard Library Trait Landscape

Rust's standard library traits fall into several categories:

```
┌─────────────────────────────────────────────────────────────────┐
│                  Standard Library Traits                        │
├──────────────┬──────────────┬───────────────┬──────────────────┤
│  Formatting  │  Comparison  │  Conversion   │    Markers       │
├──────────────┼──────────────┼───────────────┼──────────────────┤
│  Display     │  PartialEq   │  From / Into  │  Copy            │
│  Debug       │  Eq          │  TryFrom      │  Send            │
│  Binary      │  PartialOrd  │  TryInto      │  Sync            │
│  Octal       │  Ord         │  AsRef/AsMut  │  Sized           │
│  LowerHex    │  Hash        │  Deref        │  Unpin           │
│  UpperHex    │              │  DerefMut     │                  │
├──────────────┼──────────────┼───────────────┼──────────────────┤
│  Cloning     │  Iteration   │  Destruction  │  Operators       │
├──────────────┼──────────────┼───────────────┼──────────────────┤
│  Clone       │  Iterator    │  Drop         │  Add, Sub, Mul   │
│  Default     │  IntoIterator│               │  Index, IndexMut │
│              │  FromIterator│               │  Fn, FnMut       │
└──────────────┴──────────────┴───────────────┴──────────────────┘
```

You don't need to memorize all of them right now. This tutorial covers the ones you'll use most often.

---

## Display & Debug — Formatting Traits

These two traits control how your types are printed.

### Debug

```rust
// Debug is for developers — used with {:?}
#[derive(Debug)]        // <-- Almost always derive this
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };
    println!("{:?}", p);    // Point { x: 1.0, y: 2.0 }
    println!("{:#?}", p);   // Pretty-printed with indentation
}
```

**Key facts about Debug:**
- Can be `#[derive]`d automatically — the compiler generates it from fields
- Used with `{:?}` (compact) and `{:#?}` (pretty-printed)
- Required by many standard library features (e.g., `assert_eq!` needs Debug for error messages)
- **Rule of thumb:** Put `#[derive(Debug)]` on every type you define

### Display

```rust
use std::fmt;

// Display is for users — used with {}
struct Point {
    x: f64,
    y: f64,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };
    println!("{}", p);      // (1.0, 2.0)
    
    // Display also gives you .to_string() for free!
    let s: String = p.to_string();  // "(1.0, 2.0)"
}
```

**Key facts about Display:**
- **Cannot** be derived — you must implement it manually
- Used with `{}` format specifier
- Implementing Display automatically gives you `ToString` (via a blanket impl)
- Meant for end-user output — should be clean and readable

### Other Formatting Traits

| Trait | Format Specifier | Example Output |
|-------|:----------------:|----------------|
| `Debug` | `{:?}` | `Point { x: 1.0, y: 2.0 }` |
| `Display` | `{}` | `(1.0, 2.0)` |
| `Binary` | `{:b}` | `1010` |
| `Octal` | `{:o}` | `12` |
| `LowerHex` | `{:x}` | `ff` |
| `UpperHex` | `{:X}` | `FF` |
| `LowerExp` | `{:e}` | `3.14e0` |
| `UpperExp` | `{:E}` | `3.14E0` |

---

## Clone & Copy — Duplication Traits

### Clone

Clone creates an **explicit deep copy** of a value:

```rust
#[derive(Clone, Debug)]
struct Config {
    name: String,
    values: Vec<i32>,
}

fn main() {
    let original = Config {
        name: String::from("default"),
        values: vec![1, 2, 3],
    };
    
    let copy = original.clone();  // Deep copy — new heap allocations
    
    println!("{:?}", original);   // Still valid
    println!("{:?}", copy);       // Independent copy
}
```

**Clone key facts:**
- Can be derived with `#[derive(Clone)]`
- `.clone()` is always **explicit** — you see it in the code
- Can be expensive (allocates new heap memory for String, Vec, etc.)
- Custom impl when you need special cloning logic

### Copy

Copy enables **implicit bitwise duplication**:

```rust
#[derive(Copy, Clone, Debug)]  // Copy requires Clone
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let a = Point { x: 1.0, y: 2.0 };
    let b = a;       // COPIES, not moves! a is still valid
    
    println!("{:?}", a);  // Works! a wasn't moved
    println!("{:?}", b);  // Independent copy
}
```

**Copy key facts:**
- Marker trait — no methods, just signals "bitwise copy is safe"
- Requires `Clone` (Copy is a subtrait of Clone)
- Only for types that live entirely on the stack (no heap data)
- Implemented by: all integer types, floats, bool, char, tuples/arrays of Copy types, shared references `&T`

### The Drop-Copy Exclusion

**A type CANNOT implement both Copy and Drop.** Why?

- Copy means "duplicate by copying bits" — simple memcpy
- Drop means "run cleanup code when destroyed" — which copy runs the destructor?
- If both existed, dropping one copy might invalidate the other

```
Copy types:  i32, f64, bool, (i32, i32), [u8; 4]
Drop types:  String, Vec<T>, File, Box<T>
             ↑ These manage heap resources
```

---

## PartialEq & Eq — Equality Traits

### PartialEq

Enables `==` and `!=` operators:

```rust
#[derive(PartialEq, Debug)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let a = Point { x: 1.0, y: 2.0 };
    let b = Point { x: 1.0, y: 2.0 };
    let c = Point { x: 3.0, y: 4.0 };
    
    assert!(a == b);     // true — PartialEq compares field by field
    assert!(a != c);     // true
}
```

You can also compare between different types:

```rust
// PartialEq<String> for &str is implemented in std
assert!("hello" == String::from("hello"));
```

### Eq

`Eq` is a **marker trait** that extends `PartialEq` with one additional guarantee:

> **Reflexivity:** `a == a` is ALWAYS `true`

```rust
#[derive(PartialEq, Eq)]  // Eq has no additional methods
struct UserId(u64);
```

### The NaN Problem

Why have **two** equality traits? Because of floating-point NaN:

```rust
fn main() {
    let nan = f64::NAN;
    
    assert!(nan != nan);        // true! NaN is not equal to itself
    assert!(!(nan == nan));     // also true!
}
```

This violates reflexivity (`a == a` should be true), so:

| Type | `PartialEq` | `Eq` |
|------|:-----------:|:----:|
| `i32`, `u64`, etc. | ✅ | ✅ |
| `String`, `&str` | ✅ | ✅ |
| `f32`, `f64` | ✅ | ❌ — NaN breaks reflexivity |
| `bool` | ✅ | ✅ |

**Practical impact:** You can't use `f64` as a `HashMap` key (requires `Eq + Hash`).

---

## PartialOrd & Ord — Ordering Traits

### PartialOrd

Enables `<`, `>`, `<=`, `>=` and provides `partial_cmp()`:

```rust
use std::cmp::Ordering;

fn main() {
    let a = 5;
    let b = 10;
    
    // partial_cmp returns Option<Ordering>
    match a.partial_cmp(&b) {
        Some(Ordering::Less) => println!("a < b"),
        Some(Ordering::Equal) => println!("a == b"),
        Some(Ordering::Greater) => println!("a > b"),
        None => println!("Cannot compare"),  // e.g., NaN vs anything
    }
}
```

### Ord — Total Ordering

`Ord` guarantees that **any two values** can be compared — no `None` result:

```rust
fn main() {
    let mut numbers = vec![3, 1, 4, 1, 5, 9];
    numbers.sort();  // sort() requires Ord
    // [1, 1, 3, 4, 5, 9]
}
```

| Type | `PartialOrd` | `Ord` |
|------|:------------:|:-----:|
| Integers | ✅ | ✅ |
| `String` | ✅ | ✅ (lexicographic) |
| `f64` | ✅ | ❌ — NaN is incomparable |

**Practical impact:** You can't sort a `Vec<f64>` with `.sort()` — use `.sort_by()` with a custom comparator instead:

```rust
let mut floats = vec![3.0, 1.0, f64::NAN, 2.0];
floats.sort_by(|a, b| a.partial_cmp(b).unwrap_or(std::cmp::Ordering::Equal));
```

---

## Hash — For HashMaps and HashSets

The `Hash` trait lets a value be hashed for use as a key:

```rust
use std::collections::HashMap;

#[derive(Hash, PartialEq, Eq)]  // All three needed for HashMap keys
struct StudentId(u32);

fn main() {
    let mut grades: HashMap<StudentId, char> = HashMap::new();
    grades.insert(StudentId(1001), 'A');
    grades.insert(StudentId(1002), 'B');
}
```

**The Hash-Eq Contract:**

> If `a == b` (PartialEq), then `hash(a) == hash(b)` (Hash) **MUST** be true.

The reverse is NOT required — different values CAN have the same hash (collisions).

**Custom Hash example** — hash only by one field:

```rust
use std::hash::{Hash, Hasher};

struct Person {
    id: u64,
    name: String,    // Don't include in hash
    age: u32,        // Don't include in hash
}

impl Hash for Person {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);  // Only hash by ID
    }
}

impl PartialEq for Person {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id   // Must be consistent with Hash!
    }
}
impl Eq for Person {}
```

---

## Default — Sensible Starting Values

`Default` provides a "default" value for a type:

```rust
#[derive(Default, Debug)]
struct Config {
    host: String,       // Default: ""
    port: u16,          // Default: 0
    verbose: bool,      // Default: false
    retries: Option<u32>, // Default: None
}

fn main() {
    let config = Config::default();
    println!("{:?}", config);
    // Config { host: "", port: 0, verbose: false, retries: None }
    
    // struct update syntax — override just some fields:
    let custom = Config {
        port: 8080,
        verbose: true,
        ..Default::default()
    };
}
```

**Standard Default values:**

| Type | Default Value |
|------|:------------:|
| Integer types | `0` |
| Float types | `0.0` |
| `bool` | `false` |
| `char` | `'\0'` |
| `String` | `""` (empty) |
| `Vec<T>` | `[]` (empty) |
| `Option<T>` | `None` |
| `HashMap<K,V>` | `{}` (empty) |

Custom implementation:

```rust
impl Default for Config {
    fn default() -> Self {
        Config {
            host: String::from("localhost"),
            port: 3000,
            verbose: false,
            retries: Some(3),
        }
    }
}
```

---

## From & Into — Infallible Conversions

### From — Convert Into This Type

```rust
// From is already implemented in std for many types:
let s: String = String::from("hello");        // &str → String
let n: i64 = i64::from(42_i32);              // i32 → i64 (widening)
let v: Vec<u8> = Vec::from([1, 2, 3]);       // array → Vec

// Implementing your own:
struct Millimeters(f64);
struct Meters(f64);

impl From<Meters> for Millimeters {
    fn from(m: Meters) -> Self {
        Millimeters(m.0 * 1000.0)
    }
}

fn main() {
    let m = Meters(1.5);
    let mm = Millimeters::from(m);  // 1500.0
}
```

### Into — The Reverse Direction

When you implement `From<T> for U`, you get `Into<U> for T` **for free**:

```rust
fn main() {
    let m = Meters(1.5);
    let mm: Millimeters = m.into();  // Same conversion, different syntax
}
```

**The golden rule for function parameters**: Accept `Into<YourType>` to be flexible:

```rust
fn set_name(name: impl Into<String>) {
    let name: String = name.into();
    // ...
}

// Now both of these work:
set_name("hello");                    // &str → String
set_name(String::from("hello"));     // String → String (no-op)
```

---

## TryFrom & TryInto — Fallible Conversions

When conversion might fail:

```rust
use std::convert::TryFrom;

fn main() {
    // i32 → u8 might overflow!
    let big: i32 = 300;
    
    match u8::try_from(big) {
        Ok(val) => println!("Converted: {}", val),
        Err(e) => println!("Overflow: {}", e),  // This runs: 300 > 255
    }
    
    // Works for small values:
    let small: i32 = 42;
    let byte: u8 = u8::try_from(small).unwrap();  // Ok(42)
}
```

Custom `TryFrom`:

```rust
#[derive(Debug)]
struct Percentage(u8);

impl TryFrom<i32> for Percentage {
    type Error = String;
    
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if (0..=100).contains(&value) {
            Ok(Percentage(value as u8))
        } else {
            Err(format!("{} is not a valid percentage (0-100)", value))
        }
    }
}
```

---

## AsRef & AsMut — Cheap Reference Conversions

`AsRef<T>` converts a reference to `&T` cheaply (no allocation):

```rust
use std::path::Path;

// This function accepts String, &str, PathBuf, &Path — anything AsRef<Path>
fn print_path(path: &impl AsRef<Path>) {
    println!("Path: {}", path.as_ref().display());
}

fn main() {
    print_path(&"hello.txt");                    // &str
    print_path(&String::from("hello.txt"));      // String
    print_path(&std::path::PathBuf::from("/tmp")); // PathBuf
}
```

**Why this is useful:** Write one function that accepts many string/path types.

---

## Deref & DerefMut — Smart Pointer Magic

`Deref` enables "deref coercion" — automatic conversion through `*`:

```rust
use std::ops::Deref;

fn main() {
    let s = String::from("hello");
    
    // String implements Deref<Target = str>
    // So &String automatically coerces to &str:
    greet(&s);  // &String → &str automatically!
}

fn greet(name: &str) {
    println!("Hello, {}!", name);
}
```

**Deref coercion chain:**

```
&String  →  &str         (String: Deref<Target = str>)
&Vec<T>  →  &[T]         (Vec<T>: Deref<Target = [T]>)
&Box<T>  →  &T           (Box<T>: Deref<Target = T>)
&Rc<T>   →  &T           (Rc<T>: Deref<Target = T>)
&Arc<T>  →  &T           (Arc<T>: Deref<Target = T>)
```

This is why you can pass `&String` to functions expecting `&str`, and `&Vec<i32>` to functions expecting `&[i32]`. The compiler inserts the deref calls automatically.

---

## Iterator — The Sequence Trait

Quick recap (covered fully in Stage 6):

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    
    // 75+ provided methods: map, filter, fold, sum, collect, ...
}
```

Implement `next()`, get the entire iterator toolkit for free. This is the most impactful trait in Rust's standard library.

---

## Drop — The Destructor

```rust
struct DatabaseConnection {
    url: String,
}

impl Drop for DatabaseConnection {
    fn drop(&mut self) {
        println!("Closing connection to {}", self.url);
        // Cleanup logic here
    }
}

fn main() {
    let conn = DatabaseConnection { url: String::from("postgres://...") };
    // ... use conn ...
}  // conn.drop() called automatically here
```

**Key facts:**
- Called automatically when a value goes out of scope (RAII)
- You **cannot** call `.drop()` directly — use `std::mem::drop(value)` instead
- Drop order: struct fields in declaration order, local variables in **reverse** declaration order
- Cannot implement both `Drop` and `Copy`

---

## Send & Sync — Thread Safety Markers

These are **auto traits** — the compiler implements them automatically when safe:

| Trait | Meaning | Example |
|-------|---------|---------|
| `Send` | Safe to **move** to another thread | Most types are Send |
| `Sync` | Safe to **share references** across threads | Most types are Sync |
| NOT Send | `Rc<T>` — reference count is not atomic | Use `Arc<T>` instead |
| NOT Sync | `Cell<T>`, `RefCell<T>` — interior mutability not thread-safe | Use `Mutex<T>` instead |

```rust
use std::thread;

fn needs_send<T: Send>(val: T) {}

fn main() {
    let s = String::from("hello");
    needs_send(s);  // ✅ String is Send
    
    // let rc = std::rc::Rc::new(42);
    // needs_send(rc);  // ❌ Rc is NOT Send
}
```

**You rarely implement these manually.** They're covered in depth in Stage 12 (Concurrency).

---

## The Trait Cheat Sheet

| Trait | Purpose | Derivable? | Required For |
|-------|---------|:----------:|-------------|
| `Debug` | Developer-facing formatting `{:?}` | ✅ | `assert_eq!`, error messages |
| `Display` | User-facing formatting `{}` | ❌ | `to_string()`, `println!("{}")` |
| `Clone` | Explicit deep copy | ✅ | Duplicating owned data |
| `Copy` | Implicit bitwise copy | ✅ | Stack-only types |
| `PartialEq` | `==` and `!=` | ✅ | Comparisons, `assert_eq!` |
| `Eq` | Reflexive equality (a == a) | ✅ | `HashMap` keys |
| `PartialOrd` | `<`, `>`, `<=`, `>=` | ✅ | Comparisons |
| `Ord` | Total ordering | ✅ | `.sort()`, `BTreeMap` keys |
| `Hash` | Hash computation | ✅ | `HashMap`/`HashSet` keys |
| `Default` | Default value | ✅ | `..Default::default()` |
| `From`/`Into` | Infallible conversion | ❌ | Type conversions |
| `TryFrom`/`TryInto` | Fallible conversion | ❌ | Checked conversions |
| `AsRef`/`AsMut` | Cheap ref conversion | ❌ | Flexible API parameters |
| `Deref`/`DerefMut` | Smart pointer coercion | ❌ | Smart pointers, newtypes |
| `Iterator` | Lazy sequences | ❌ | `for` loops, `map`/`filter` |
| `Drop` | Destructor / cleanup | ❌ | Resource cleanup (RAII) |
| `Send` | Thread-safe move | Auto | `thread::spawn` |
| `Sync` | Thread-safe sharing | Auto | `Arc<T>`, shared state |

### Cross-Language Comparison

| Concept | Rust | Java | Python | C++ | Go |
|---------|------|------|--------|-----|----|
| Debug printing | `Debug` trait | `toString()` | `__repr__` | `operator<<` | `fmt.Stringer` |
| Display | `Display` | `toString()` | `__str__` | `operator<<` | `fmt.Stringer` |
| Clone | `Clone` trait | `.clone()` (Cloneable) | `copy.deepcopy` | Copy ctor | No built-in |
| Equality | `PartialEq`/`Eq` | `.equals()` | `__eq__` | `operator==` | `==` (built-in) |
| Ordering | `Ord` | `Comparable` | `__lt__` | `operator<` | `sort.Interface` |
| Hash | `Hash` | `hashCode()` | `__hash__` | `std::hash` | Not needed |
| Default | `Default` | Constructors | `__init__` | Default ctor | Zero values |

---

## Exercises

### Exercise 1: Deriving the Essentials

Create a `Color` struct with `r`, `g`, `b` fields (all `u8`). Derive all the traits needed to:
1. Print it with `{:?}`
2. Compare two colors with `==`
3. Use it as a `HashMap` key
4. Clone it
5. Copy it

<details>
<summary>💡 Solution</summary>

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

fn main() {
    let red = Color { r: 255, g: 0, b: 0 };
    let also_red = red;  // Copy, not move
    
    assert_eq!(red, also_red);       // PartialEq + Eq
    println!("{:?}", red);            // Debug
    
    let mut names: HashMap<Color, &str> = HashMap::new();
    names.insert(red, "red");         // Hash + Eq
    
    let cloned = red.clone();         // Clone
}
```

</details>

### Exercise 2: Implement Display

Create a `Temperature` enum with `Celsius(f64)` and `Fahrenheit(f64)` variants. Implement `Display` to print as "25.0°C" or "77.0°F".

<details>
<summary>💡 Solution</summary>

```rust
use std::fmt;

enum Temperature {
    Celsius(f64),
    Fahrenheit(f64),
}

impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Temperature::Celsius(t) => write!(f, "{:.1}°C", t),
            Temperature::Fahrenheit(t) => write!(f, "{:.1}°F", t),
        }
    }
}

fn main() {
    let temp = Temperature::Celsius(25.0);
    println!("{}", temp);  // 25.0°C
}
```

</details>

### Exercise 3: From Conversions

Implement `From<Celsius>` for `Fahrenheit` and vice versa (formula: F = C × 9/5 + 32):

<details>
<summary>💡 Solution</summary>

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Self {
        Celsius((f.0 - 32.0) * 5.0 / 9.0)
    }
}

fn main() {
    let boiling = Celsius(100.0);
    let f: Fahrenheit = boiling.into();  // 212.0
    println!("{}°F", f.0);
    
    let body_temp = Fahrenheit(98.6);
    let c: Celsius = body_temp.into();   // 37.0
    println!("{:.1}°C", c.0);
}
```

</details>

---

## Summary

| Category | Traits | Key Takeaway |
|----------|--------|-------------|
| Formatting | `Debug`, `Display` | Debug is derivable; Display is manual |
| Cloning | `Clone`, `Copy` | Clone = explicit deep; Copy = implicit bitwise |
| Equality | `PartialEq`, `Eq` | Eq adds reflexivity; f64 is NOT Eq |
| Ordering | `PartialOrd`, `Ord` | Ord = total ordering; needed for sort() |
| Hashing | `Hash` | Must be consistent with Eq |
| Defaults | `Default` | Sensible zero/empty values |
| Conversion | `From`/`Into`, `TryFrom`/`TryInto` | From = infallible, TryFrom = fallible |
| References | `AsRef`, `Deref` | Cheap conversions and smart pointer magic |
| Lifecycle | `Drop` | RAII — automatic cleanup |
| Threading | `Send`, `Sync` | Auto-traits for thread safety |

**Rule of thumb:** Start every struct with `#[derive(Debug, Clone)]`. Add `PartialEq, Eq, Hash` when needed for collections. Add `Copy` only for small, stack-only types.

---

**Previous:** [← Trait Bounds](./05-trait-bounds.md) · **Next:** [Operator Overloading →](./07-operator-overloading.md)

<p align="center"><i>Tutorial 6 of 9 — Stage 8: Generics & Traits</i></p>

# Clone & Copy 🧬

> **Move semantics prevent accidental copies. But sometimes you genuinely NEED a copy. Rust gives you two traits for that: `Clone` (explicit deep copy) and `Copy` (implicit cheap copy).**

---

## Table of Contents

- [The Problem: Sometimes You Need Two Copies](#the-problem-sometimes-you-need-two-copies)
- [Clone — Explicit Deep Copy](#clone--explicit-deep-copy)
  - [How Clone Works](#how-clone-works)
  - [Clone in Memory](#clone-in-memory)
  - [When to Use Clone](#when-to-use-clone)
  - [The Cost of Clone](#the-cost-of-clone)
  - [Cloning Nested Types](#cloning-nested-types)
- [Copy — Implicit Cheap Copy](#copy--implicit-cheap-copy)
  - [How Copy Works](#how-copy-works)
  - [Types That Implement Copy](#types-that-implement-copy)
  - [Types That Do NOT Implement Copy](#types-that-do-not-implement-copy)
  - [Copy in Memory](#copy-in-memory)
  - [Why Copy Is Implicit](#why-copy-is-implicit)
- [Clone vs Copy — The Full Picture](#clone-vs-copy--the-full-picture)
- [Deriving Clone and Copy](#deriving-clone-and-copy)
  - [derive(Clone)](#deriveclone)
  - [derive(Copy, Clone)](#derivecopy-clone)
  - [When You Can't Derive Copy](#when-you-cant-derive-copy)
- [Custom Clone Implementations](#custom-clone-implementations)
- [Common Patterns](#common-patterns)
- [Clone Performance Tips](#clone-performance-tips)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Problem: Sometimes You Need Two Copies

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // Move — s1 is invalid
    
    // But what if I need BOTH s1 and s2?
    // What if I want to send a copy to a function AND keep one?
}
```

Rust's answer: use `.clone()` to explicitly create a deep copy, or rely on `Copy` for cheap stack types.

---

## Clone — Explicit Deep Copy

The `Clone` trait provides the `.clone()` method, which creates a **full, independent copy** of a value — including any heap data.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // Deep copy — s1 is STILL valid!
    
    println!("s1 = {}", s1);  // ✅
    println!("s2 = {}", s2);  // ✅
    
    // s1 and s2 are completely independent
}
```

### How Clone Works

When you call `.clone()`:

1. **New heap memory is allocated** (same size as the original)
2. **All bytes are copied** from the original heap memory to the new one
3. **A new stack value is created** pointing to the new heap memory
4. **The original is untouched** — it still works

### Clone in Memory

```rust
let s1 = String::from("hello");
let s2 = s1.clone();
```

```
Before clone:

Stack                    Heap
┌──────────────┐        ┌───┬───┬───┬───┬───┐
│ s1:          │        │ h │ e │ l │ l │ o │
│   ptr ───────│───────→│   │   │   │   │   │
│   len: 5     │        └───┴───┴───┴───┴───┘
│   cap: 5     │        Address: 0x1000
└──────────────┘

After clone:

Stack                    Heap
┌──────────────┐        ┌───┬───┬───┬───┬───┐
│ s1:          │        │ h │ e │ l │ l │ o │
│   ptr ───────│───────→│   │   │   │   │   │  ← Original (0x1000)
│   len: 5     │        └───┴───┴───┴───┴───┘
│   cap: 5     │
├──────────────┤        ┌───┬───┬───┬───┬───┐
│ s2:          │        │ h │ e │ l │ l │ o │
│   ptr ───────│───────→│   │   │   │   │   │  ← NEW copy (0x2000)
│   len: 5     │        └───┴───┴───┴───┴───┘
│   cap: 5     │
└──────────────┘

Two separate heap allocations!
Modifying one does NOT affect the other.
```

### Proving They're Independent

```rust
fn main() {
    let mut s1 = String::from("hello");
    let mut s2 = s1.clone();
    
    s1.push_str(" world");
    s2.push_str(" rust");
    
    println!("s1 = {}", s1);  // "hello world"
    println!("s2 = {}", s2);  // "hello rust"
    // Completely independent — modifying one doesn't affect the other
}
```

### When to Use Clone

```rust
// 1. When a function takes ownership but you need to keep your copy
fn process(data: String) {
    println!("Processing: {}", data);
}

fn main() {
    let original = String::from("important data");
    process(original.clone());  // Send a copy
    println!("Still have: {}", original);  // Keep the original
}

// 2. When you need the same data in multiple places
fn main() {
    let config = String::from("debug=true");
    let logger_config = config.clone();
    let server_config = config.clone();
    // All three are independent copies
}

// 3. When you want to modify a copy without affecting the original
fn main() {
    let template = String::from("Hello, {name}!");
    let mut greeting = template.clone();
    greeting = greeting.replace("{name}", "Alice");
    
    println!("Template: {}", template);   // "Hello, {name}!"
    println!("Greeting: {}", greeting);   // "Hello, Alice!"
}
```

### The Cost of Clone

**Clone is NOT free.** It allocates heap memory and copies bytes:

```
Clone cost for String of length N:
  1. Allocate N bytes on heap       → ~25-100 ns
  2. Copy N bytes                   → O(N) — proportional to size
  3. Create new stack metadata      → negligible

Total: O(N) — grows with the size of the data

Compare:
  Move: O(1) — always 24 bytes, regardless of data size
  Clone: O(N) — proportional to the data
```

For a tiny string ("hi"): clone is cheap.  
For a 10 MB string: clone copies 10 MB. Think before you clone!

### Cloning Nested Types

Clone is **deep** — it recursively clones all owned data:

```rust
fn main() {
    let original = vec![
        String::from("hello"),
        String::from("world"),
    ];
    
    let cloned = original.clone();
    // This clones:
    // 1. The Vec (new heap array for pointers)
    // 2. Each String in the Vec (new heap allocation for each)
    // Total: 3 new heap allocations!
    
    println!("{:?}", original);  // ["hello", "world"]
    println!("{:?}", cloned);    // ["hello", "world"]
}
```

```
original (stack)           Heap
┌──────────────┐          ┌────────────────┐
│ ptr ─────────│─────────→│ ptr→"hello"    │ → heap: "hello"
│ len: 2       │          │ ptr→"world"    │ → heap: "world"
│ cap: 2       │          └────────────────┘
└──────────────┘

cloned (stack)             Heap
┌──────────────┐          ┌────────────────┐
│ ptr ─────────│─────────→│ ptr→"hello"    │ → heap: "hello" (NEW COPY)
│ len: 2       │          │ ptr→"world"    │ → heap: "world" (NEW COPY)
│ cap: 2       │          └────────────────┘

Everything is duplicated — 3 heap allocations for the clone!
```

---

## Copy — Implicit Cheap Copy

The `Copy` trait marks types that can be duplicated by simply copying their bytes. The copy happens **automatically** (implicitly) — you don't call `.clone()`.

### How Copy Works

```rust
fn main() {
    let x: i32 = 42;
    let y = x;  // COPY happens automatically — x is still valid!
    
    println!("x = {}", x);  // ✅
    println!("y = {}", y);  // ✅
}
```

No `.clone()` needed. The assignment `let y = x;` implicitly copies the value.

### Types That Implement Copy

All types whose data is **entirely on the stack** and can be duplicated by copying bytes:

```rust
// ✅ All of these implement Copy

// Integers
let a: i8 = 1;
let b: i16 = 2;
let c: i32 = 3;
let d: i64 = 4;
let e: i128 = 5;
let f: isize = 6;
let g: u8 = 7;
let h: u16 = 8;
let i: u32 = 9;
let j: u64 = 10;
let k: u128 = 11;
let l: usize = 12;

// Floats
let m: f32 = 1.0;
let n: f64 = 2.0;

// Boolean
let o: bool = true;

// Character
let p: char = 'A';

// Tuples of Copy types
let q: (i32, f64) = (1, 2.0);
let r: (bool, char, i32) = (true, 'x', 42);

// Arrays of Copy types
let s: [i32; 5] = [1, 2, 3, 4, 5];

// References (shared only!)
let val = 42;
let t: &i32 = &val;
```

### Types That Do NOT Implement Copy

Any type that **owns heap data** or manages a resource:

```rust
// ❌ These do NOT implement Copy

let a: String = String::from("hello");     // Owns heap data
let b: Vec<i32> = vec![1, 2, 3];          // Owns heap data
let c: Box<i32> = Box::new(42);           // Owns heap data
// HashMap, BTreeMap, etc.                  // Own heap data
// File, TcpStream, etc.                   // Manage OS resources
```

Why not? Because copying them would require duplicating heap memory, which is expensive and should be explicit (`.clone()`).

### Copy in Memory

Copy literally means "copy the bytes":

```rust
let x: i32 = 42;
let y = x;  // Copy
```

```
Stack:
┌────────────────┐
│ y: 42 (i32)    │  ← Byte-for-byte copy of x
├────────────────┤
│ x: 42 (i32)    │  ← Original
└────────────────┘

Two independent copies, each 4 bytes.
No heap involved. Cost: copying 4 bytes ≈ instant.
```

Compare with a tuple:

```rust
let t1: (i32, f64, bool) = (42, 3.14, true);
let t2 = t1;  // Copy — all fields are Copy
```

```
Stack:
┌───────────────────────────┐
│ t2: (42, 3.14, true)      │  ← Copy (13 bytes + padding)
├───────────────────────────┤
│ t1: (42, 3.14, true)      │  ← Original
└───────────────────────────┘

Both independent. No heap. Instant.
```

### Why Copy Is Implicit

Copying stack data is so cheap that making it explicit (requiring `.clone()` every time) would be annoying:

```rust
// If i32 required explicit clone:
let x: i32 = 1;
let y = x.clone();  // Tedious!
let z = x.clone();  // Even more tedious!
println!("{} {} {}", x.clone(), y.clone(), z.clone());  // AWFUL!

// With implicit Copy:
let x: i32 = 1;
let y = x;           // Clean
let z = x;           // Clean
println!("{} {} {}", x, y, z);  // Clean!
```

The rule: **if copying is always cheap and safe, make it implicit.**

---

## Clone vs Copy — The Full Picture

| Feature | `Clone` | `Copy` |
|---------|---------|--------|
| **How to use** | `.clone()` (explicit) | Automatic (implicit) |
| **What it does** | Deep copy (may allocate heap) | Byte-for-byte copy (stack only) |
| **Performance** | Can be expensive (O(N)) | Always cheap (O(1), small N) |
| **Types** | Most types can implement it | Only stack-only types |
| **Heap allocation?** | Usually yes | Never |
| **Original after?** | Still valid | Still valid |
| **Trait relationship** | Standalone | `Copy` requires `Clone` |

```
              ┌───────────────┐
              │     Clone     │  ← Most types implement this
              │  (explicit)   │
              │  .clone()     │
              └───────┬───────┘
                      │ subset
              ┌───────┴───────┐
              │     Copy      │  ← Only cheap, stack-only types
              │  (implicit)   │
              │  auto-copy    │
              └───────────────┘
```

**Every `Copy` type is also `Clone`**, but not every `Clone` type is `Copy`. If a type is `Copy`, you can still call `.clone()` — it just does the same thing as the implicit copy.

---

## Deriving Clone and Copy

### `derive(Clone)`

You can automatically implement `Clone` for your own types using `#[derive]`:

```rust
#[derive(Clone, Debug)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = p1.clone();
    
    println!("p1 = {:?}", p1);  // ✅ Point { x: 1.0, y: 2.0 }
    println!("p2 = {:?}", p2);  // ✅ Point { x: 1.0, y: 2.0 }
}
```

The derive macro automatically clones each field:

```rust
// What #[derive(Clone)] generates (conceptually):
impl Clone for Point {
    fn clone(&self) -> Self {
        Point {
            x: self.x.clone(),  // f64 clone (just copies)
            y: self.y.clone(),  // f64 clone (just copies)
        }
    }
}
```

### `derive(Copy, Clone)`

To make a type `Copy`, ALL fields must also be `Copy`. You **must** also derive `Clone` (since `Copy` requires `Clone`):

```rust
#[derive(Copy, Clone, Debug)]
struct Point {
    x: f64,    // f64 is Copy ✅
    y: f64,    // f64 is Copy ✅
}

fn main() {
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = p1;  // COPY — p1 is still valid!
    
    println!("p1 = {:?}", p1);  // ✅
    println!("p2 = {:?}", p2);  // ✅
}
```

### When You Can't Derive Copy

If ANY field is not `Copy`, the whole struct can't be `Copy`:

```rust
#[derive(Clone, Debug)]
struct User {
    name: String,  // String is NOT Copy
    age: i32,      // i32 IS Copy
}

// #[derive(Copy)]  // ❌ ERROR: String doesn't implement Copy!

fn main() {
    let u1 = User { name: String::from("Alice"), age: 30 };
    // let u2 = u1;  // This would be a MOVE (not copy)
    let u2 = u1.clone();  // Must explicitly clone
    
    println!("{:?}", u1);  // ✅ (because we cloned, not moved)
    println!("{:?}", u2);  // ✅
}
```

### Decision Tree: Should My Type Be Copy?

```
Does my type contain any heap data?
  │
  ├── YES (String, Vec, Box, HashMap, etc.)
  │   → Cannot be Copy. Derive Clone instead.
  │
  └── NO (all fields are i32, f64, bool, etc.)
      │
      Is my type small? (<= 64 bytes or so?)
        │
        ├── YES
        │   → Derive Copy and Clone
        │
        └── NO (e.g., [f64; 1000])
            → Consider only Clone
            (Large copies could be surprisingly expensive)
```

---

## Custom Clone Implementations

Sometimes the derived `Clone` isn't what you want. You can implement it manually:

```rust
#[derive(Debug)]
struct Counter {
    name: String,
    count: u64,
}

impl Clone for Counter {
    fn clone(&self) -> Self {
        Counter {
            name: self.name.clone(),
            count: 0,  // Reset count on clone!
        }
    }
}

fn main() {
    let c1 = Counter { name: String::from("hits"), count: 42 };
    let c2 = c1.clone();
    
    println!("c1 = {:?}", c1);  // Counter { name: "hits", count: 42 }
    println!("c2 = {:?}", c2);  // Counter { name: "hits", count: 0 }
}
```

---

## Common Patterns

### Pattern 1: Clone Before a Consuming Function

```rust
fn send_message(msg: String) {
    println!("Sent: {}", msg);
}

fn main() {
    let msg = String::from("Hello!");
    
    send_message(msg.clone());  // Send a copy
    send_message(msg.clone());  // Send another copy
    send_message(msg);          // Send the original (last use, no need to clone)
}
```

### Pattern 2: Clone Into Collections

```rust
fn main() {
    let base_name = String::from("User");
    let mut names = Vec::new();
    
    for i in 1..=3 {
        let mut name = base_name.clone();
        name.push_str(&format!("_{}", i));
        names.push(name);
    }
    
    println!("{:?}", names);  // ["User_1", "User_2", "User_3"]
    println!("Base: {}", base_name);  // Still valid!
}
```

### Pattern 3: Copy Types in Functions

```rust
fn square(n: i32) -> i32 {
    n * n
}

fn main() {
    let x = 5;
    let a = square(x);  // x is copied — still valid
    let b = square(x);  // Copied again
    println!("x={}, a={}, b={}", x, a, b);
}
```

### Pattern 4: to_owned() — Clone for Borrowed Data

```rust
fn main() {
    let s: &str = "hello";       // s is a borrowed string slice
    let owned: String = s.to_owned();  // Creates an owned String
    // .to_owned() is essentially clone for borrowed → owned conversion
    
    let v: &[i32] = &[1, 2, 3];       // v is a borrowed slice
    let owned_v: Vec<i32> = v.to_owned();  // Creates an owned Vec
    
    println!("{}", owned);      // "hello"
    println!("{:?}", owned_v);  // [1, 2, 3]
}
```

---

## Clone Performance Tips

### 1. Don't Clone When a Reference Will Do

```rust
fn print_name(name: &String) {  // Borrow instead of clone
    println!("{}", name);
}

fn main() {
    let name = String::from("Alice");
    print_name(&name);  // ✅ No clone needed — just borrow
    print_name(&name);  // ✅ Can borrow as many times as you want
}
```

### 2. Clone Once, Not in a Loop

```rust
// ❌ Cloning in every iteration
for item in &data {
    let copy = expensive_value.clone();  // N clones!
    process(copy, item);
}

// ✅ Clone once outside the loop (if possible)
let copy = expensive_value.clone();  // 1 clone
for item in &data {
    process(&copy, item);
}
```

### 3. Move Instead of Clone When You're Done With the Original

```rust
// ❌ Unnecessary clone
let data = create_big_data();
let result = process(data.clone());
// data is never used again! The clone was wasted.

// ✅ Move instead
let data = create_big_data();
let result = process(data);  // Move — no copy needed
```

### 4. Avoid Cloning in `unwrap_or`

```rust
let default = String::from("default");
let maybe_val: Option<String> = None;

// ❌ Clones default even when maybe_val is Some
let val = maybe_val.unwrap_or(default.clone());

// ✅ unwrap_or_else only clones when needed
let maybe_val: Option<String> = None;
let val = maybe_val.unwrap_or_else(|| default.clone());
```

---

## Common Mistakes

### Mistake 1: Excessive Cloning

"When in doubt, just `.clone()` everything" — this is a common beginner strategy. It works, but creates unnecessary performance overhead:

```rust
// ❌ "Clone all the things!"
fn process(data: String) -> String {
    data.clone()  // Why clone? Just return data directly!
}

// ✅ No clone needed — just return the owned value
fn process(data: String) -> String {
    data
}
```

### Mistake 2: Trying to Derive Copy on a Type with String

```rust
// ❌ Won't compile
// #[derive(Copy, Clone)]
// struct User {
//     name: String,  // String is not Copy!
//     age: i32,
// }

// ✅ Only derive Clone
#[derive(Clone)]
struct User {
    name: String,
    age: i32,
}
```

### Mistake 3: Forgetting That Clone Creates Independent Copies

```rust
let mut v1 = vec![1, 2, 3];
let v2 = v1.clone();

v1.push(4);

println!("{:?}", v1);  // [1, 2, 3, 4]
println!("{:?}", v2);  // [1, 2, 3]  ← NOT [1, 2, 3, 4]!
// They're independent — modifying one doesn't affect the other
```

### Mistake 4: Confusing Copy and Clone Semantics

```rust
// Copy: happens automatically on assignment
let x: i32 = 42;
let y = x;  // Copy (implicit)
// x and y are both valid, both equal 42

// Clone: must be called explicitly for heap types
let s1 = String::from("hello");
let s2 = s1;  // MOVE, not clone!
// s1 is invalid! To clone: let s2 = s1.clone();
```

---

## Exercises

### Exercise 1: Clone or Move?

For each assignment, state whether it's a move, copy, or you need to add `.clone()`:

```rust
let a: i32 = 42;
let b = a;              // (?)

let c = String::from("hello");
let d = c;              // (?)

let e: f64 = 3.14;
let f = e;              // (?)

let g = vec![1, 2, 3];
let h = g;              // (?)

let i: (i32, bool) = (1, true);
let j = i;              // (?)

let k: (String, i32) = (String::from("hi"), 42);
let l = k;              // (?)
```

<details>
<summary>Answer</summary>

| Variable | What Happens | Why |
|----------|-------------|-----|
| `b = a` | **Copy** | `i32` is Copy |
| `d = c` | **Move** | `String` is not Copy |
| `f = e` | **Copy** | `f64` is Copy |
| `h = g` | **Move** | `Vec<i32>` is not Copy |
| `j = i` | **Copy** | Tuple of (i32, bool) — all fields are Copy |
| `l = k` | **Move** | Tuple contains String, which is not Copy |

</details>

### Exercise 2: Fix With Clone

Make this compile using `.clone()`:

```rust
fn main() {
    let names = vec!["Alice".to_string(), "Bob".to_string()];
    let backup = names;
    println!("Original: {:?}", names);
    println!("Backup: {:?}", backup);
}
```

### Exercise 3: Derive the Right Traits

Add the correct `#[derive(...)]` attributes to make this code compile:

```rust
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

struct Label {
    text: String,
    color: Color,
}

fn main() {
    let red = Color { r: 255, g: 0, b: 0 };
    let red2 = red;   // Should copy
    println!("{} {} {}", red.r, red.g, red.b);  // red should still work
    
    let label = Label { text: "Hello".to_string(), color: red2 };
    let label2 = label.clone();  // Should clone
    println!("{}", label.text);   // label should still work
}
```

<details>
<summary>Answer</summary>

```rust
#[derive(Copy, Clone)]  // All fields (u8) are Copy
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

#[derive(Clone)]  // String is not Copy, so Label can't be Copy
struct Label {
    text: String,
    color: Color,
}
```

</details>

### Exercise 4: Minimize Clones

Rewrite this code to use the minimum number of `.clone()` calls (ideally zero, using moves and references):

```rust
fn process(data: String) {
    println!("Processing: {}", data);
}

fn main() {
    let data = String::from("important");
    process(data.clone());
    process(data.clone());
    process(data.clone());
}
```

---

## Summary

### Clone vs Copy

| | `Clone` | `Copy` |
|---|---------|--------|
| **Syntax** | `value.clone()` | Automatic on assignment |
| **Speed** | Can be slow (heap allocation) | Always fast (stack bytes) |
| **Requirement** | Any type can implement it | All fields must be Copy |
| **Heap types** | ✅ String, Vec, etc. | ❌ Not allowed |
| **Stack types** | ✅ Works too | ✅ Preferred |
| **Relationship** | Standalone trait | Subset of Clone |

### Which Types Are Copy?

| Copy ✅ | Not Copy ❌ |
|---------|------------|
| `i8`..`i128`, `u8`..`u128` | `String` |
| `f32`, `f64` | `Vec<T>` |
| `bool` | `Box<T>` |
| `char` | `HashMap<K,V>` |
| `&T` | `&mut T` |
| `(Copy, Copy, ...)` tuples | Tuples with non-Copy |
| `[Copy; N]` arrays | Structs with non-Copy fields |

### Key Takeaways

1. **Clone = explicit deep copy** — you must call `.clone()`; it can be expensive
2. **Copy = implicit bitwise copy** — happens automatically; always cheap
3. **Every `Copy` type is also `Clone`**, but not vice versa
4. **Don't overuse `.clone()`** — prefer references (`&T`) when possible
5. **`Copy` requires all fields to be `Copy`** — one `String` field prevents it
6. **Derive `Copy, Clone` for small, stack-only types** like `Point { x: f64, y: f64 }`
7. **Derive `Clone` only for types with heap data** like `struct User { name: String }`
8. **Move when you can, borrow when you should, clone as a last resort**

---

## What's Next?

We've been talking about how moving and cloning transfer ownership. But what if you want to let a function USE a value without taking ownership? That's what **references** and **borrowing** are for.

**Next Tutorial:** [References & Borrowing →](./05-references-and-borrowing.md)

---

<p align="center">
  <i>Tutorial 4 of 8 — Stage 3: Ownership</i>
</p>

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

### The Copy Trait — What Makes a Type Trivially Copyable

At the deepest level, a type is `Copy` if and only if it can be **safely duplicated by copying its raw bytes** — a `memcpy` in C terms. The C++ standard calls this property `std::is_trivially_copyable`: the object's representation in memory IS the object's value, with no external resources or indirection.

**The Formal Rule: Drop and Copy Are Mutually Exclusive**

If a type implements `Drop` (a destructor), it **cannot** be `Copy`. Why? Consider what would happen:

```
Suppose String were Copy:

    let s1 = String::from("hello");    // s1 owns heap buffer at 0x1000
    let s2 = s1;                        // Bitwise copy → s2 ALSO points to 0x1000
                                        // Now TWO owners of the same heap buffer!
    
    // End of scope:
    //   drop(s2) → frees 0x1000
    //   drop(s1) → frees 0x1000 AGAIN → double free! 💥
```

The `Drop` trait means "this type needs cleanup logic." The `Copy` trait means "duplicating my bytes creates a valid, independent value." These two promises are inherently contradictory for types that manage resources — if the destructor frees a resource, a bitwise copy would create two values that both try to free the same resource.

**What's Actually Copy and Why:**

```
    Copy Types (all stack-resident, no destructor):
    ┌─────────────────────────────────────────────────────┐
    │  Primitives:  i8..i128, u8..u128, f32, f64,        │
    │               bool, char                            │
    │  Pointers:    &T (shared reference), *const T,      │
    │               *mut T, fn pointers                   │
    │  Composites:  (A, B) if A: Copy and B: Copy         │
    │               [T; N] if T: Copy                     │
    └─────────────────────────────────────────────────────┘

    NOT Copy (owns heap data or has Drop):
    ┌─────────────────────────────────────────────────────┐
    │  String, Vec<T>, Box<T>, HashMap<K,V>,              │
    │  Rc<T>, Arc<T>, File, TcpStream, MutexGuard<T>,     │
    │  &mut T (would violate borrowing rules!)            │
    └─────────────────────────────────────────────────────┘
```

**Why `&T` Is Copy but `&mut T` Is NOT:**

A shared reference `&T` is just a pointer (8 bytes on 64-bit). Duplicating those bytes gives you two shared references to the same data — perfectly fine, since Rust allows unlimited shared references.  
But `&mut T` is an **exclusive** reference. If you could copy it, you'd have two exclusive references to the same data, violating Rust's core borrowing invariant: *either* one `&mut` *or* many `&`, never both.

**The Copy Trait Is a Zero-Cost Abstraction:**

When you assign `let y = x;` for a `Copy` type, the compiler emits the exact same machine code as C's raw assignment — a `memcpy` of N bytes (or often just a register-to-register `mov`). There is no trait method call, no vtable lookup, no overhead whatsoever. Copy types are as fast as bare metal.

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

### Shallow Copy vs Deep Copy — The Universal Problem

The distinction between "shallow copy" and "deep copy" isn't unique to Rust — it's a fundamental problem in **every** programming language that has heap-allocated or reference-based data. What makes Rust special is how it encodes this distinction into the **type system** at compile time, rather than leaving it as a runtime footgun.

**Python — The Mutable Surprise:**

```python
import copy

original = [[1, 2], [3, 4]]
shallow  = original.copy()          # or list(original)
deep     = copy.deepcopy(original)

original[0].append(999)

print(shallow)  # [[1, 2, 999], [3, 4]]  ← OOPS! Shared inner lists!
print(deep)     # [[1, 2], [3, 4]]        ← Safe, fully independent
```

Python's `list.copy()` only copies the **top-level list object**. The inner lists are still shared references. This is the textbook shallow-copy bug, and it has caused countless production incidents.

**JavaScript — No Built-In Deep Copy (Until 2022):**

```javascript
const obj = { a: 1, nested: { b: 2 } };
const shallow = { ...obj };          // spread is shallow
shallow.nested.b = 999;
console.log(obj.nested.b);           // 999 — original mutated!

// Hacky deep copy before 2022:
const deep = JSON.parse(JSON.stringify(obj));  // Breaks on Date, RegExp, etc.

// Modern deep copy (2022+):
const deep2 = structuredClone(obj);            // Finally correct!
```

JavaScript developers spent years using `JSON.parse(JSON.stringify(...))` as a "deep copy" — a hack that silently drops functions, `undefined` values, `Date` objects, and circular references.

**Java — `Cloneable` Is Considered Broken:**

Java's `Object.clone()` performs a **shallow copy** by default. Joshua Bloch (author of *Effective Java*) calls `Cloneable` "a deeply broken interface" because:
- It doesn't declare a `clone()` method (the method is on `Object`)
- Deep copy requires manually cloning every mutable field
- It's easy to forget a field and introduce shared-mutable-state bugs

**C++ — Copy Constructors Default to Shallow:**

C++ generates a default copy constructor that copies each member — but for raw pointers, this copies the **pointer value**, not the pointed-to data. This leads to double-free bugs when both copies try to deallocate the same memory. The "Rule of Three" (destructor + copy constructor + copy assignment) exists because of this problem.

**Rust's Insight: Encode the Distinction in the Type System**

| Language   | Shallow Copy Syntax        | Deep Copy Syntax              | Default Behavior    | Safety          |
|------------|----------------------------|-------------------------------|---------------------|-----------------|
| Python     | `list.copy()`, `[:]`       | `copy.deepcopy()`             | Assignment shares   | Runtime bugs    |
| JavaScript | `{...obj}`, `Object.assign`| `structuredClone()`           | Assignment shares   | Runtime bugs    |
| Java       | `clone()` (default)        | Manual recursive clone        | Assignment shares   | Runtime bugs    |
| C++        | Default copy constructor   | Custom copy constructor       | Shallow copy        | Double-free UB  |
| **Rust**   | `Copy` (implicit bitwise)  | `Clone` (explicit `.clone()`) | **Move** (no copy!) | **Compile-time safe** |

Rust's key innovation: the **default is neither shallow nor deep copy — it's a move**. Copies only happen when you explicitly opt in (`Clone`) or when the type is proven safe for implicit bitwise duplication (`Copy`). The compiler enforces this, so shallow-copy bugs are structurally impossible.

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

### Performance: When to Clone and When to Borrow

Clone is a correctness escape hatch, not a performance strategy. Understanding its cost — and knowing the alternatives — is the difference between Rust code that flies and Rust code that crawls.

**The Real Cost of Cloning:**

```
Operation               | What happens                          | Approximate cost
------------------------|---------------------------------------|-------------------
String::clone() (1 KB)  | malloc(1024) + memcpy(1024 bytes)     | ~100 ns
String::clone() (1 MB)  | malloc(1M) + memcpy(1M bytes)         | ~50 μs
Vec<T>::clone() (1000)  | malloc + clone each of 1000 elements  | O(n × clone(T))
HashMap::clone()        | malloc + rehash + clone all entries   | O(n × (clone(K)+clone(V)))
&T (borrowing)          | Copy a pointer (8 bytes)              | ~0.3 ns
```

Borrowing is roughly **150,000×** faster than cloning a 1 MB String. That's not a typo.

**The "Clone Tax" Anti-Pattern:**

Beginners often sprinkle `.clone()` everywhere to appease the borrow checker. This is sometimes called the "clone tax":

```rust
// ❌ The clone tax — works, but wasteful
fn analyze(data: &Vec<String>) -> Vec<String> {
    let mut results = Vec::new();
    for item in data {
        let owned = item.clone();       // Cloning every element!
        if owned.starts_with("important") {
            results.push(owned);
        }
    }
    results
}

// ✅ Better: borrow and only clone what you keep
fn analyze(data: &[String]) -> Vec<String> {
    data.iter()
        .filter(|s| s.starts_with("important"))
        .cloned()                        // Clone only the matches
        .collect()
}
```

**The Rule of Thumb:** If you're cloning inside a loop, step back and ask if you can restructure to use references instead.

**Alternatives to Cloning:**

```
┌───────────────────────────────────────────────────────────────┐
│  Instead of...          │  Consider...                       │
├───────────────────────────────────────────────────────────────┤
│  data.clone()           │  &data (borrow it)                 │
│  clone for shared use   │  Rc<T> / Arc<T> (shared ownership) │
│  clone to maybe-mutate  │  Cow<T> (clone on write)           │
│  clone across threads   │  Arc<T> + Mutex<T> or channels     │
│  clone in last usage    │  Just move it!                     │
└───────────────────────────────────────────────────────────────┘
```

**`Cow<T>` — Clone on Write (The Best of Both Worlds):**

`Cow` (Clone on Write) starts as a borrow and only allocates when you need to mutate:

```rust
use std::borrow::Cow;

fn maybe_uppercase(s: &str, shout: bool) -> Cow<str> {
    if shout {
        Cow::Owned(s.to_uppercase())  // Allocates only when needed
    } else {
        Cow::Borrowed(s)              // Zero cost — just a reference
    }
}

fn main() {
    let quiet = maybe_uppercase("hello", false);  // No allocation
    let loud  = maybe_uppercase("hello", true);   // Allocates "HELLO"
    println!("{quiet}, {loud}");  // "hello, HELLO"
}
```

**When Cloning IS the Right Call:**

- **Small, immutable config values** — a 20-byte string is cheaper to clone than to manage lifetimes across your entire program
- **Thread boundaries** — when sending data to another thread, cloning (or `Arc`) is often necessary since references can't cross thread boundaries without lifetime gymnastics
- **Prototyping** — clone freely while designing, then optimize with borrows and `Rc`/`Arc` once the design stabilizes
- **The hot-path rule** — if profiling shows the clone isn't in a hot loop, the clarity of owned data may outweigh the nanoseconds saved

The golden rule: **profile before you optimize**. A clone that runs once at startup is irrelevant. A clone inside a per-frame game loop processing 60,000 entities will destroy your frame rate.

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

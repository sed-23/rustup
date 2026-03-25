# Move Semantics 🚚

> **When you assign one variable to another in Rust, you're not copying — you're MOVING. The original variable becomes invalid. This is how Rust prevents double-free bugs at zero cost.**

---

## Table of Contents

- [What Is a Move?](#what-is-a-move)
- [Moves in Memory — Step by Step](#moves-in-memory--step-by-step)
  - [The Shallow Copy Problem](#the-shallow-copy-problem)
  - [Rust's Solution: Move = Shallow Copy + Invalidate](#rusts-solution-move--shallow-copy--invalidate)
- [Where Moves Happen](#where-moves-happen)
  - [Variable Assignment](#variable-assignment)
  - [Function Parameters](#function-parameters)
  - [Function Return Values](#function-return-values)
  - [Pushing Into Collections](#pushing-into-collections)
  - [Match and if let](#match-and-if-let)
- [What Gets Moved and What Gets Copied](#what-gets-moved-and-what-gets-copied)
- [The Compiler Error: "value used here after move"](#the-compiler-error-value-used-here-after-move)
  - [Reading the Error Message](#reading-the-error-message)
  - [Fixing Move Errors](#fixing-move-errors)
- [Moves and Control Flow](#moves-and-control-flow)
  - [Moves in if/else](#moves-in-ifelse)
  - [Moves in Loops](#moves-in-loops)
- [Partial Moves](#partial-moves)
- [Move Semantics vs Other Languages](#move-semantics-vs-other-languages)
- [Why Moves Are Good](#why-moves-are-good)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is a Move?

A **move** is the transfer of ownership from one variable to another. After a move:

- The **new** variable owns the data
- The **old** variable is **invalid** and cannot be used
- No data is duplicated on the heap — only the stack metadata is copied

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // ← This is a MOVE
    
    // s1 is now invalid
    // s2 now owns "hello"
    
    println!("{}", s2);  // ✅ Works
    // println!("{}", s1);  // ❌ Compile error
}
```

---

## Moves in Memory — Step by Step

To understand moves, you first need to understand what would go wrong WITHOUT them.

### The Shallow Copy Problem

Recall that a `String` looks like this in memory:

```
Stack                    Heap
┌──────────────┐        ┌───┬───┬───┬───┬───┐
│ ptr ─────────│───────→│ h │ e │ l │ l │ o │
│ len: 5       │        └───┴───┴───┴───┴───┘
│ capacity: 5  │
└──────────────┘
```

What if `let s2 = s1` just copied the stack data (a **shallow copy**)?

```
s1 (stack)               Heap                    s2 (stack)
┌──────────────┐        ┌───┬───┬───┬───┬───┐  ┌──────────────┐
│ ptr ─────────│────┐   │ h │ e │ l │ l │ o │  │ ptr ─────────│────┐
│ len: 5       │    └──→│   │   │   │   │   │←─┘ len: 5       │    │
│ capacity: 5  │        └───┴───┴───┴───┴───┘  │ capacity: 5  │    │
└──────────────┘                                └──────────────┘    │
                               ↑                                    │
                               └────────────────────────────────────┘
                         BOTH point to the same memory!
```

Now when `s1` and `s2` go out of scope, BOTH try to free the same heap memory. **That's a double free** — undefined behavior, crashes, security holes.

### Rust's Solution: Move = Shallow Copy + Invalidate

Rust copies the stack data (pointer, length, capacity) to `s2`, then **invalidates** `s1`:

```
Step 1: Before move
                                 Heap
s1 (stack)                      ┌───┬───┬───┬───┬───┐
┌──────────────┐               │ h │ e │ l │ l │ o │
│ ptr ─────────│──────────────→│   │   │   │   │   │
│ len: 5       │               └───┴───┴───┴───┴───┘
│ capacity: 5  │
└──────────────┘

Step 2: let s2 = s1  (MOVE)
                                 Heap
s1 (INVALID)                    ┌───┬───┬───┬───┬───┐
┌──────────────┐               │ h │ e │ l │ l │ o │
│ ████████████ │         ┌────→│   │   │   │   │   │
│ ████████████ │         │     └───┴───┴───┴───┴───┘
│ ████████████ │         │
└──────────────┘         │
                          │
s2 (stack — new owner)   │
┌──────────────┐         │
│ ptr ─────────│─────────┘
│ len: 5       │
│ capacity: 5  │
└──────────────┘
```

what actually happens at the machine level:
1. **24 bytes are copied** from `s1`'s stack slot to `s2`'s stack slot (pointer + len + cap)
2. The compiler **remembers** that `s1` is invalid
3. The heap data at `"hello"` **doesn't move** — only the pointer was copied
4. When `s2` goes out of scope → heap memory freed **once** ✅
5. `s1` is never dropped because it's invalid — **no double free** ✅

> **Key insight:** A "move" in Rust is NOT physically moving data. It's copying 24 bytes of stack metadata and invalidating the source. It's as cheap as a shallow copy.

### What Actually Happens in Memory During a Move

Let's go even deeper than the diagrams above. At the **machine level**, a move is just a `memcpy` — a
bitwise copy of bytes from one stack location to another. The magic is entirely in the **compiler**,
not the CPU.

#### The Exact Byte Layout

A `String` on a 64-bit system occupies exactly **24 bytes** on the stack:

```
Offset  Field       Size     Example Value
──────  ──────────  ───────  ─────────────────────
 0..8   ptr         8 bytes  0x00007f3a_bc001a20
 8..16  len         8 bytes  0x00000000_00000005
16..24  capacity    8 bytes  0x00000000_00000008
```

When you write `let s2 = s1;`, the CPU copies these **24 bytes** from `s1`'s stack
slot to `s2`'s stack slot. That's it. The heap buffer at address `0x00007f3a_bc001a20`
containing `"hello"` is **never touched**.

```
Before move:
  Stack addr 0x7ffd1000: [ptr=0x7f3abc001a20, len=5, cap=8]  ← s1

After move (memcpy of 24 bytes):
  Stack addr 0x7ffd1000: [ptr=0x7f3abc001a20, len=5, cap=8]  ← s1 (INVALID — compiler marks it)
  Stack addr 0x7ffd1018: [ptr=0x7f3abc001a20, len=5, cap=8]  ← s2 (new owner)

Heap (unchanged!):
  Addr 0x7f3abc001a20: [104, 101, 108, 108, 111]  ← "hello" in UTF-8
```

#### Why Moves Are O(1)

This is a critical performance insight: **the cost of a move is always the size of the
stack metadata, never the size of the heap data.** Moving a `String` containing 1 byte
costs the same as moving a `String` containing 1 GB — exactly 24 bytes copied.

```
Data Size    Deep Copy Cost    Move Cost
─────────    ──────────────    ─────────
10 bytes     ~34 bytes         24 bytes
1 KB         ~1048 bytes       24 bytes
1 MB         ~1,048,600 bytes  24 bytes
1 GB         ~1.07 billion     24 bytes   ← Same!
```

#### How LLVM Optimizes: Moves That Vanish

In practice, many moves compile to **zero instructions**. LLVM applies several optimizations:

- **Named Return Value Optimization (NRVO):** When a function returns a value, the compiler
  may construct it directly in the caller's stack frame, skipping the move entirely
- **Copy elision:** When the compiler can prove the source is unused after the move, it
  reuses the same stack slot
- **Register allocation:** Small types that "move" may live entirely in CPU registers —
  the move is just the compiler reassigning which register maps to which variable name

```rust
fn make_string() -> String {
    let s = String::from("hello");  // Constructed directly in caller's frame
    s  // "Move" is optimized away — no memcpy at all in release mode
}
```

#### Contrast with C++ Move Constructors

In C++, a move is defined by a **user-written move constructor** that can do arbitrary work.
A C++ `std::move` might allocate, log, lock a mutex, or even copy data — the language
provides no guarantees. In Rust, moves are **always** a fixed-size `memcpy` (or nothing at
all after optimization). There's no user-defined move logic, which is why moves are
predictable and free.

---

## Where Moves Happen

### Variable Assignment

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // MOVE
    
    // println!("{}", s1);  // ❌ s1 is invalid
    println!("{}", s2);     // ✅
}
```

### Function Parameters

Passing a value to a function moves ownership to the parameter:

```rust
fn eat_string(s: String) {
    println!("Eating: {}", s);
}  // s is dropped here

fn main() {
    let dinner = String::from("pizza");
    eat_string(dinner);  // MOVE: dinner → s
    
    // println!("{}", dinner);  // ❌ dinner was moved
}
```

```
Before call:     dinner ──→ "pizza"
During call:     dinner ✗   s ──→ "pizza"
After call:      dinner ✗   "pizza" FREED
```

### Function Return Values

Returning a value moves ownership to the caller:

```rust
fn make_greeting() -> String {
    let s = String::from("Hello!");
    s  // MOVE: s → (return value) → caller
}  // s is NOT dropped — it was moved out

fn main() {
    let greeting = make_greeting();  // greeting now owns "Hello!"
    println!("{}", greeting);
}  // greeting is dropped here
```

### Pushing Into Collections

```rust
fn main() {
    let mut names = Vec::new();
    
    let name = String::from("Alice");
    names.push(name);  // MOVE: name → into the Vec
    
    // println!("{}", name);  // ❌ name was moved into names
    println!("{:?}", names);  // ✅ ["Alice"]
}
```

### Match and `if let`

```rust
fn main() {
    let maybe_name: Option<String> = Some(String::from("Bob"));
    
    match maybe_name {
        Some(name) => println!("Found: {}", name),  // name takes ownership
        None => println!("No name"),
    }
    
    // println!("{:?}", maybe_name);  // ❌ moved in the match
}
```

To avoid this, match on a reference:

```rust
fn main() {
    let maybe_name: Option<String> = Some(String::from("Bob"));
    
    match &maybe_name {
        Some(name) => println!("Found: {}", name),  // name is &String
        None => println!("No name"),
    }
    
    println!("{:?}", maybe_name);  // ✅ still valid
}
```

---

## What Gets Moved and What Gets Copied

Not all types move! Types that implement the `Copy` trait are **copied** instead:

| Type | Move or Copy? | Why? |
|------|--------------|------|
| `i8`, `i16`, `i32`, `i64`, `i128`, `isize` | **Copy** | Small, stack-only, cheap to copy |
| `u8`, `u16`, `u32`, `u64`, `u128`, `usize` | **Copy** | Small, stack-only, cheap to copy |
| `f32`, `f64` | **Copy** | Small, stack-only |
| `bool` | **Copy** | 1 byte |
| `char` | **Copy** | 4 bytes |
| `(i32, f64)` (tuples of Copy types) | **Copy** | All fields are Copy |
| `[i32; 5]` (arrays of Copy types) | **Copy** | All elements are Copy |
| `&T` (shared references) | **Copy** | Just a pointer |
| `String` | **Move** | Owns heap data |
| `Vec<T>` | **Move** | Owns heap data |
| `Box<T>` | **Move** | Owns heap data |
| `(String, i32)` | **Move** | Contains a non-Copy type |

The rule: **If a type owns heap memory, it moves. If it's entirely on the stack and cheap to duplicate, it copies.**

```rust
fn main() {
    // Copy types — both original and copy are valid
    let x = 42;
    let y = x;      // COPY (x is still valid)
    println!("x={}, y={}", x, y);  // ✅ Both valid
    
    // Move types — only the new owner is valid
    let s1 = String::from("hello");
    let s2 = s1;    // MOVE (s1 is invalid)
    // println!("{}", s1);  // ❌
    println!("{}", s2);     // ✅
}
```

### Why the Distinction?

Copying an `i32` means copying 4 bytes on the stack. That's trivial.

Copying a `String` would mean allocating NEW heap memory and copying ALL the bytes. For a 1 MB string, that's a 1 MB copy just from an assignment. Rust doesn't do that automatically — it moves instead. If you explicitly want a deep copy, use `.clone()` (next tutorial).

```
i32 copy: Copy 4 bytes on stack → instant, negligible
String "copy": Allocate + copy N bytes on heap → expensive!
String move: Copy 24 bytes on stack + invalidate old → cheap!
```

---

## The Compiler Error: "value used here after move"

This is the most common ownership error. Let's learn to read it:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    println!("{}", s1);
}
```

```
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:4:20
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`,
  |            which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 |     println!("{}", s1);
  |                    ^^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let s2 = s1.clone();
  |                ++++++++
```

### Reading the Error Message

The compiler tells you:
1. **What happened:** "borrow of moved value: `s1`" — you tried to use `s1` after it was moved
2. **Why it moved:** "`s1` has type `String`, which does not implement the `Copy` trait" — String doesn't auto-copy
3. **Where it moved:** "value moved here" at `let s2 = s1;`
4. **Where you tried to use it:** "value borrowed here after move" at `println!("{}", s1)`
5. **How to fix it:** "consider cloning the value" — `s1.clone()`

### Fixing Move Errors

There are several strategies:

**Strategy 1: Use the new owner**
```rust
let s1 = String::from("hello");
let s2 = s1;
println!("{}", s2);  // Use s2 instead of s1
```

**Strategy 2: Clone before moving**
```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // Deep copy — s1 is still valid
println!("{} {}", s1, s2);
```

**Strategy 3: Use a reference (borrow)**
```rust
let s1 = String::from("hello");
let s2 = &s1;  // Borrow — s1 keeps ownership
println!("{} {}", s1, s2);
```

**Strategy 4: Restructure code to avoid the move**
```rust
fn process(s: &String) {  // Take a reference instead
    println!("{}", s);
}

let s1 = String::from("hello");
process(&s1);  // Lend, don't give
println!("{}", s1);  // Still valid
```

---

## Moves and Control Flow

### Moves in `if/else`

The compiler analyses BOTH branches:

```rust
fn main() {
    let s = String::from("hello");
    let condition = true;
    
    if condition {
        let s2 = s;  // s is moved here...
        println!("{}", s2);
    } else {
        let s3 = s;  // ...OR here (but not both)
        println!("{}", s3);
    }
    
    // println!("{}", s);  // ❌ s MIGHT have been moved
    // The compiler can't be sure which branch ran, so it
    // conservatively says s is invalid after the if/else
}
```

### Moves in Loops

This is a very common surprise:

```rust
fn main() {
    let s = String::from("hello");
    
    // for _ in 0..3 {
    //     let s2 = s;  // ❌ s is moved on first iteration
    //                   // Second iteration tries to move it again!
    //     println!("{}", s2);
    // }
    
    // Fix: clone each iteration
    for _ in 0..3 {
        let s2 = s.clone();  // ✅ Creates a new copy each iteration
        println!("{}", s2);
    }
    
    // Or: use a reference
    for _ in 0..3 {
        println!("{}", &s);  // ✅ Just borrows
    }
}
```

---

## Partial Moves

When you destructure a struct or tuple, you can move **some** fields while keeping others:

```rust
fn main() {
    let pair = (String::from("hello"), 42);
    
    let (s, n) = pair;
    // s takes ownership of the String (move)
    // n copies the i32 (copy)
    
    // pair is now partially moved
    // println!("{}", pair.0);  // ❌ String was moved  
    // println!("{}", pair.1);  // ❌ Can't use partial pair either
    
    println!("{} {}", s, n);  // ✅
}
```

With named structs:

```rust
struct Person {
    name: String,
    age: i32,
}

fn main() {
    let person = Person {
        name: String::from("Alice"),
        age: 30,
    };
    
    let name = person.name;  // Moves name out of person
    
    println!("Name: {}", name);        // ✅
    println!("Age: {}", person.age);   // ✅ (i32 is Copy)
    // println!("{}", person.name);    // ❌ name was moved out
}
```

---

## Move Semantics vs Other Languages

### C++: Copy by Default

In C++, `std::string s2 = s1;` creates a **deep copy** — allocating new heap memory and copying all the bytes. This is safe but can be expensive.

C++11 added `std::move(s1)` to opt-in to move semantics. But unlike Rust, using `s1` after `std::move` is **undefined behavior**, not a compile error.

```cpp
// C++
std::string s1 = "hello";
std::string s2 = s1;            // Deep copy (expensive!)
std::string s3 = std::move(s1); // Move (cheap, but s1 is now "valid but unspecified")
std::cout << s1;                // ← Compiles but UNDEFINED BEHAVIOR!
```

### Java/Python: Everything is a Reference

In Java and Python, `s2 = s1` just copies a reference — both point to the same object. A garbage collector handles cleanup.

```python
# Python
s1 = "hello"
s2 = s1   # Both reference the same string object
print(s1)  # Works fine
print(s2)  # Same thing
# GC frees the string when both s1 and s2 are gone
```

### Rust: Move by Default (for heap types)

```rust
// Rust
let s1 = String::from("hello");
let s2 = s1;  // Move (cheap — 24 bytes copied, s1 invalidated)
// println!("{}", s1);  // ❌ COMPILE ERROR — caught before runtime!
println!("{}", s2);     // ✅
```

The comparison:

| Language | `s2 = s1` for Strings | Cost | Safety |
|----------|----------------------|------|--------|
| C++ | Deep copy (default) | Expensive | Safe (but wasteful) |
| C++ (std::move) | Move | Cheap | Unsafe if you use s1 |
| Java/Python | Reference copy | Cheap | Safe (GC handles it) |
| **Rust** | **Move** | **Cheap** | **Safe (compiler enforced)** |

Rust gets the best of both worlds: the cheapness of a move AND compile-time safety.

### Move Semantics Across Programming Languages

Rust's move semantics didn't emerge in a vacuum. Every language handles the question
"what happens when I assign one variable to another?" differently. Understanding the
full spectrum helps you appreciate what Rust chose — and why.

#### C: No Moves — Just Raw Memory Copies

C has no concept of "moves." Assignment is always a bitwise copy of the struct's bytes.
If a struct contains a pointer, both the original and the copy point to the same memory.
The programmer is entirely responsible for tracking who should free it.

```c
// C — assignment copies the struct, including the pointer
struct MyString { char* data; size_t len; };
struct MyString s1 = make_string("hello");
struct MyString s2 = s1;  // Bitwise copy — now BOTH have the same data pointer
free(s1.data);             // s2.data is now a DANGLING POINTER
printf("%s", s2.data);    // 💥 Use-after-free
```

#### C++: Moves Bolted On in C++11

C++ defaulted to deep copies for decades. In C++11, **rvalue references** (`&&`) and
`std::move` were added to opt into move semantics. This was a major language evolution,
but it requires significant boilerplate:

```cpp
// C++ — you must write a move constructor manually
class MyString {
public:
    char* data;
    size_t len;

    // Move constructor — YOU define what "move" means
    MyString(MyString&& other) noexcept
        : data(other.data), len(other.len) {
        other.data = nullptr;  // Must manually null out the source
        other.len = 0;
    }
};

MyString s1 = make_string("hello");
MyString s2 = std::move(s1);  // Calls YOUR move constructor
// s1 is now in a "valid but unspecified" state — NOT a compile error to use it!
std::cout << s1.data;  // 💥 Probably nullptr dereference, but COMPILES FINE
```

Compare this to Rust where moves are **implicit, require zero boilerplate, and the
compiler prevents using the moved-from variable.** C++ gave programmers the tool;
Rust made it the default.

#### Java & Python: References All the Way Down

In Java and Python, variables are **always** references to objects on the heap.
"Moving" doesn't exist because you're only ever copying a pointer:

```java
// Java — s2 = s1 copies the reference, not the object
String s1 = new String("hello");
String s2 = s1;  // Both are references to the same object
// The garbage collector frees the object when nothing references it
```

This is simple, but it means: no single ownership, no deterministic destruction,
and the GC adds runtime overhead and pause-time unpredictability.

#### Swift: Learning From Rust

Swift 5.9 (2023) introduced the `consuming` keyword, directly inspired by Rust.
It allows functions to take ownership of a value, preventing the caller from using
it afterward — following the same principle Rust has had since 1.0.

#### The Key Insight

Rust made moves the **DEFAULT** behavior for non-trivial types, not an optimization
requiring special syntax. Every other language either doesn't have moves (C, Java,
Python), bolted them on later (C++), or is adopting them now (Swift).

| Language | Default Assignment | Move Support            | Use-After-Move Safety |
|----------|-------------------|-------------------------|------------------------|
| C        | Bitwise copy      | None                    | None — undefined behavior |
| C++      | Deep copy         | Opt-in (`std::move`)    | No — compiles, UB at runtime |
| Java     | Reference copy    | N/A (GC manages memory) | N/A — no ownership concept |
| Python   | Reference copy    | N/A (GC manages memory) | N/A — no ownership concept |
| Swift    | Copy (value) / Ref (class) | `consuming` (5.9+) | Partial — compiler checks consuming |
| **Rust** | **Move**          | **Implicit (default)**  | **Full — compile-time error** |

---

## Why Moves Are Good

### 1. Performance: No Hidden Copies

```rust
// Passing a 1 MB String to a function:
fn process(data: String) { /* ... */ }

let big_data = String::from("...1 MB of text...");
process(big_data);
// ↑ This moves 24 bytes (pointer + len + cap), NOT 1 MB
// Zero-cost — no heap allocation, no data duplication
```

### 2. Clarity: You Know When Data Is Consumed

```rust
fn send_over_network(msg: String) {
    // msg is consumed — it's clear the caller can't use it anymore
}

let message = String::from("secret");
send_over_network(message);
// message is gone — you KNOW it was consumed
// No question about whether it's still in memory
```

### 3. Safety: Double-Free Is Impossible

Only one variable owns the data at any time. Only one drop happens. Double-free is structurally impossible.

### 4. Deterministic: You Know Exactly When Memory Is Freed

Unlike garbage collection (which frees "sometime later"), moves make cleanup predictable:

```rust
{
    let s = String::from("hello");
    // ... use s ...
}  // s dropped HERE — exactly at this line, every time
```

---

## Common Mistakes

### Mistake 1: Trying to Use a Moved Value

```rust
let v = vec![1, 2, 3];
let v2 = v;
// println!("{:?}", v);  // ❌ v was moved

// Fix: use v2, or clone v first
println!("{:?}", v2);  // ✅
```

### Mistake 2: Moving in a Loop

```rust
let name = String::from("Alice");

// ❌ Moves name on first iteration, fails on second
// for _ in 0..3 {
//     println!("{}", name);  // Actually this is a borrow, let's show a real move:
//     let x = name;  // ← Move happens here
// }

// ✅ Clone if you need separate copies
for _ in 0..3 {
    let x = name.clone();
    println!("{}", x);
}
```

### Mistake 3: Forgetting That Collections Own Their Elements

```rust
let mut vec = Vec::new();
let s = String::from("hello");
vec.push(s);
// println!("{}", s);  // ❌ s was moved into vec

// The Vec now owns the String
println!("{:?}", vec);  // ✅ ["hello"]
```

### Mistake 4: Assuming All Assignment Is a Move

```rust
let x = 42;
let y = x;  // This is a COPY, not a move!
println!("x={}, y={}", x, y);  // ✅ Both valid
// i32 implements Copy, so x is duplicated, not moved
```

### Partial Moves and the Pin API — Advanced Move Concepts

We've seen basic partial moves above, but the concept has deeper implications that
connect to some of Rust's most advanced features.

#### Why Partial Moves Exist

When you move one field out of a struct, Rust makes the **entire struct** inaccessible
(except for fields that weren't moved). This maintains the core guarantee: **every
value has exactly one owner at all times.**

```rust
struct Record {
    name: String,
    data: Vec<u8>,
    id: u32,
}

fn main() {
    let r = Record {
        name: String::from("Alice"),
        data: vec![1, 2, 3],
        id: 42,
    };

    let extracted_name = r.name;  // Partial move: name moved out

    println!("id: {}", r.id);          // ✅ id is Copy — still accessible
    // println!("{:?}", r.data);       // ❌ Whole struct is partially moved
    // println!("{:?}", r.name);       // ❌ name was moved
    println!("name: {}", extracted_name); // ✅
}
```

#### The Self-Referential Struct Problem

Consider a struct that contains a pointer to its **own** field:

```
struct SelfRef {
    data: String,
    ptr: *const String,  ← Points to self.data!
}

Memory layout BEFORE move:
┌─────────────────────────────┐
│ data: String (at addr 0x100)│ ←──┐
│ ptr:  0x100                 │ ───┘  Points to data above
└─────────────────────────────┘

Memory layout AFTER move to new address:
┌─────────────────────────────┐
│ data: String (at addr 0x200)│      data is now here!
│ ptr:  0x100                 │ ───→ DANGLING! Still points to old location
└─────────────────────────────┘
```

If you move this struct, the internal pointer (`ptr`) still points to the **old**
address. The data is now at a new address, but the pointer wasn't updated. This is
a dangling pointer — exactly the kind of bug Rust is designed to prevent.

**Real-world analogy:** Imagine moving a house that has internal plumbing. The pipes
connect rooms within the house. If you pick up the house and move it to a new lot, the
internal pipes still reference the old room locations — water flows to nowhere. You'd
need to re-route every pipe after the move.

#### Why `async/await` Needed `Pin<T>`

This isn't just a theoretical problem. When you write `async fn`, the compiler generates
a **state machine struct** that captures local variables across `.await` points. If one
variable is a reference to another, the struct becomes self-referential:

```rust
async fn example() {
    let data = String::from("hello");
    let reference = &data;  // reference points to data
    some_async_op().await;  // State machine captures both data AND reference
    println!("{}", reference);
}
// The generated state machine struct holds both `data` and `reference`
// where `reference` points to `data` — a self-referential struct!
```

If this state machine were moved (e.g., sent to another thread), `reference` would
dangle. This is why Rust introduced `Pin<T>`.

#### What `Pin<T>` Does

`Pin<T>` is a wrapper that **promises the inner value will not be moved** after pinning.
It doesn't change the value's memory layout — it's a compile-time contract that prevents
calling any API that would move the value.

```
Normal value:    Can be moved freely          ──→  let b = a; ✅
Pinned value:    Pin<&mut T> prevents moves   ──→  Can't move out ✅ (self-ref safe)
```

This is an advanced topic, but it shows how deep the move-semantics rabbit hole goes:
Rust's entire `async` ecosystem depends on being able to reason about whether a value
can be moved, and `Pin` is the safety valve for the cases where it can't.

> **Key takeaway:** Move semantics aren't just about assignment — they're a foundational
> concept that shapes Rust's approach to async programming, self-referential data, and
> zero-cost abstractions. Understanding moves deeply prepares you for Rust's most
> powerful features.

---

## Exercises

### Exercise 1: Predict the Output

Will each program compile? If yes, what does it print? If no, which line fails?

**Program A:**
```rust
fn main() {
    let a = String::from("hello");
    let b = a;
    let c = b;
    println!("{}", c);
}
```

**Program B:**
```rust
fn main() {
    let a = String::from("hello");
    let b = a;
    println!("{}", a);
}
```

**Program C:**
```rust
fn main() {
    let a = 10;
    let b = a;
    let c = a;
    println!("{} {} {}", a, b, c);
}
```

**Program D:**
```rust
fn greet(name: String) {
    println!("Hello, {}!", name);
}

fn main() {
    let name = String::from("World");
    greet(name);
    greet(name);
}
```

<details>
<summary>Answers</summary>

- **A:** ✅ Compiles. Prints "hello". Ownership: a → b → c.
- **B:** ❌ Compile error. `a` was moved to `b`, then used on `println!` line.
- **C:** ✅ Compiles. Prints "10 10 10". `i32` is Copy, so `a` is copied each time.
- **D:** ❌ Compile error. `name` is moved into the first `greet()` call, then the second call tries to move it again.

</details>

### Exercise 2: Fix the Broken Code

Make this compile by using `.clone()`:

```rust
fn main() {
    let msg = String::from("hello");
    let a = msg;
    let b = msg;
    let c = msg;
    println!("{} {} {}", a, b, c);
}
```

### Exercise 3: Trace the Ownership

For each line, write down who owns the String "Rust":

```rust
fn take(s: String) -> String {
    println!("Taken: {}", s);
    s
}

fn main() {
    let x = String::from("Rust");   // Line A
    let y = take(x);                 // Line B
    let z = y;                       // Line C
    println!("{}", z);               // Line D
}
```

### Exercise 4: Why Does This Work?

Explain why this code compiles without errors, even though `x` is used multiple times:

```rust
fn double(n: i32) -> i32 {
    n * 2
}

fn main() {
    let x = 5;
    let a = double(x);
    let b = double(x);
    let c = double(x);
    println!("{} {} {} {}", x, a, b, c);
}
```

---

## Summary

### What Moves Are

| Aspect | Description |
|--------|-------------|
| **What happens** | Stack metadata is copied, old variable is invalidated |
| **Heap data** | Stays in place — only the pointer changes hands |
| **Cost** | Cheap — just copies a few bytes on the stack |
| **Old variable** | Invalid — compile error if used |
| **New variable** | Now the sole owner |

### Where Moves Happen

| Context | Example |
|---------|---------|
| Variable assignment | `let s2 = s1;` |
| Function call | `process(s);` |
| Function return | `return s;` |
| Collection push | `vec.push(s);` |
| Pattern matching | `let (a, b) = tuple;` |

### Move vs Copy

| Move (Default for heap types) | Copy (for stack-only types) |
|-------------------------------|----------------------------|
| Original is invalidated | Original is still valid |
| Only one owner at a time | Both have independent copies |
| `String`, `Vec`, `Box` | `i32`, `f64`, `bool`, `char`, `&T` |
| Cheap (copy pointer only) | Cheap (copy small stack data) |

### Key Takeaways

1. **A move copies stack metadata and invalidates the source** — the heap data doesn't move
2. **Moves are cheap** — typically 8-24 bytes copied on the stack
3. **The compiler prevents use-after-move** at compile time
4. **Stack-only types (i32, bool, etc.) are copied, not moved**
5. **Passing a value to a function moves it** — the caller loses access
6. **Returning a value from a function moves it** — the caller gains ownership
7. **The compile error message tells you exactly what happened** and suggests fixes

---

## What's Next?

We know that moves invalidate the source variable. But what if you actually NEED two independent copies of the data? That's what `Clone` and `Copy` are for.

**Next Tutorial:** [Clone & Copy →](./04-clone-and-copy.md)

---

<p align="center">
  <i>Tutorial 3 of 8 — Stage 3: Ownership</i>
</p>

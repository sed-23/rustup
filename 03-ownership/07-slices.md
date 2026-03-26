# Slices 🔪

> **A slice is a reference to a contiguous sequence of elements in a collection. It lets you work with a portion of data without copying it or taking ownership.**

---

## Table of Contents

- [The Problem Slices Solve](#the-problem-slices-solve)
- [What Is a Slice?](#what-is-a-slice)
  - [Slices Are Fat Pointers](#slices-are-fat-pointers)
  - [Slice Memory Layout](#slice-memory-layout)
- [String Slices (&str)](#string-slices-str)
  - [Creating String Slices](#creating-string-slices)
  - [Range Syntax Shortcuts](#range-syntax-shortcuts)
  - [Byte Boundaries and UTF-8](#byte-boundaries-and-utf-8)
  - [String Literals Are Slices](#string-literals-are-slices)
- [Array and Vec Slices (&\[T\])](#array-and-vec-slices-t)
  - [Creating Array Slices](#creating-array-slices)
  - [Vec Slices](#vec-slices)
- [Slices Enforce Borrowing Rules](#slices-enforce-borrowing-rules)
- [Mutable Slices (&mut \[T\])](#mutable-slices-mut-t)
- [Slices in Functions](#slices-in-functions)
  - [Why Accept &\[T\] Instead of &Vec\<T\>](#why-accept-t-instead-of-vect)
  - [Why Accept &str Instead of &String](#why-accept-str-instead-of-string)
- [Common Slice Methods](#common-slice-methods)
- [Slice Patterns](#slice-patterns)
  - [Pattern 1: Find-in-Slice](#pattern-1-find-in-slice)
  - [Pattern 2: Split and Process](#pattern-2-split-and-process)
  - [Pattern 3: Windows and Chunks](#pattern-3-windows-and-chunks)
  - [Pattern 4: Slice of Slices](#pattern-4-slice-of-slices)
- [How Slices Prevent Bugs](#how-slices-prevent-bugs)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Problem Slices Solve

Imagine you want a function that returns the first word of a string. Without slices, you'd return an index:

```rust
fn first_word_index(s: &String) -> usize {
    let bytes = s.as_bytes();
    
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return i;
        }
    }
    
    s.len()  // The whole string is one word
}
```

The problem? The index can become **invalid**:

```rust
fn main() {
    let mut s = String::from("hello world");
    let word_end = first_word_index(&s);  // 5
    
    s.clear();  // Empty the string!
    
    // word_end is still 5 — but the string is empty!
    // This is a logic bug. The index is meaningless now.
    println!("First word ends at: {}", word_end);  // 5 for an empty string?!
}
```

**Slices solve this** by connecting the reference to the data it refers to, so the compiler can enforce validity.

---

## What Is a Slice?

A **slice** is a **reference to a contiguous portion** of a collection. It doesn't own the data — it just points to it.

```rust
fn main() {
    let s = String::from("hello world");
    
    let hello = &s[0..5];   // Slice of "hello"
    let world = &s[6..11];  // Slice of "world"
    
    println!("{}", hello);  // "hello"
    println!("{}", world);  // "world"
}
```

### Slices Are Fat Pointers

A regular reference (`&T`) is a single pointer (8 bytes on 64-bit). A **slice** is a **fat pointer** — it stores TWO things:

```
Regular reference (&String):
┌──────────┐
│ pointer  │  → points to the String struct
│ (8 bytes)│
└──────────┘

Slice (&str or &[T]):
┌──────────┐
│ pointer  │  → points to the start of the data
│ (8 bytes)│
├──────────┤
│ length   │  → how many elements (or bytes) in this slice
│ (8 bytes)│
└──────────┘

Total: 16 bytes (on 64-bit systems)
```

### Slice Memory Layout

```rust
let s = String::from("hello world");
let hello = &s[0..5];
let world = &s[6..11];
```

```
Stack:                          Heap:
                              ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
s: ┌────────────────┐         │ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │
   │ ptr ───────────────────→ └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
   │ len: 11        │          ↑[0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]  [8]  [9]  [10]
   │ capacity: 11   │          │                           │
   └────────────────┘          │                           │
                               │                           │
hello: ┌────────────┐          │                           │
       │ ptr ───────────────→──┘                           │
       │ len: 5     │   points to 'h' (index 0)           │
       └────────────┘                                      │
                                                           │
world: ┌────────────┐                                      │
       │ ptr ───────────────→──────────────────────────────┘
       │ len: 5     │   points to 'w' (index 6)
       └────────────┘
```

Key observations:
- `hello` and `world` don't own any heap memory
- They point **into** the same heap allocation that `s` owns
- Each slice knows its starting point and length
- No data is copied!

---

### Fat Pointers Deep Dive — How Slices Are Represented in Memory

We said slices are "fat pointers," but what does that really mean at the machine level? Let's go deeper.

#### Thin vs. Fat Pointers

A **thin pointer** (`&T`) is just a memory address — 8 bytes on a 64-bit system (one `usize`):

```
&T (thin pointer) — 8 bytes:
┌─────────────────────────────────────────────┐
│ Bytes 0-7: memory address of the T value    │
└─────────────────────────────────────────────┘
```

A **fat pointer** (`&[T]` or `&str`) carries extra metadata — 16 bytes (two `usize` values):

```
&[T] or &str (fat pointer) — 16 bytes:
┌─────────────────────────────────────────────┐
│ Bytes 0-7:  memory address of first element │
├─────────────────────────────────────────────┤
│ Bytes 8-15: number of elements (length)     │
└─────────────────────────────────────────────┘
```

You can verify this in Rust:

```rust
use std::mem::size_of;

fn main() {
    println!("&i32:      {} bytes", size_of::<&i32>());        // 8
    println!("&[i32]:    {} bytes", size_of::<&[i32]>());      // 16
    println!("&str:      {} bytes", size_of::<&str>());        // 16
    println!("&String:   {} bytes", size_of::<&String>());     // 8 (thin!)
}
```

Notice that `&String` is a **thin** pointer — it points to the `String` struct on the stack, which itself contains the pointer, length, and capacity. But `&str` is **fat** — it carries the pointer and length directly.

#### Other Fat Pointers in Rust

Trait objects are also fat pointers, but their second word is different:

```
&dyn Trait (fat pointer) — 16 bytes:
┌─────────────────────────────────────────────┐
│ Bytes 0-7:  pointer to the concrete data    │
├─────────────────────────────────────────────┤
│ Bytes 8-15: pointer to the vtable           │
└─────────────────────────────────────────────┘
```

So Rust has exactly two kinds of fat pointers: **slices** (data + length) and **trait objects** (data + vtable).

#### Why Not Store Length Inside the Data?

Historically, languages tried other approaches to know where a string (or sequence) ends:

- **C strings** use a null terminator (`'\0'`). To find the length, you walk the entire string — O(n). If the terminator is missing, you get a **buffer overflow**, one of the most exploited vulnerabilities in software history.
- **Pascal strings** stored the length in the first byte. Fast O(1) lookup, but strings were limited to **255 characters** — a hard ceiling that became a real problem.
- **Rust slices** store the length alongside the pointer in a fat pointer. O(1) length access, no maximum size (limited only by `usize::MAX`), and impossible to go out of bounds undetected.

The evolution looks like this:

```
C strings (1972)          →  Pascal strings (1970)    →  Fat pointers (modern)
null-terminated              length-prefixed              pointer + length
O(n) length                  O(1) length                  O(1) length
buffer overflow risk         255 char limit               no size limit
no bounds checking           limited bounds checking      full bounds checking
```

| Approach | Length Access | Max Size | Bounds Check | Memory Overhead |
|----------|--------------|----------|-------------|----------------|
| C strings (null-terminated) | O(n) scan | Unlimited | None (unsafe) | 1 byte (the `\0`) |
| Pascal strings (length prefix) | O(1) | 255 bytes | Possible | 1 byte (length) |
| Rust fat pointers | O(1) | `usize::MAX` | Guaranteed | 8 bytes (length in pointer) |

Rust's approach costs an extra 8 bytes per slice on the stack, but that's a trivial price for safety and speed. You never scan for a terminator, you never overflow a buffer, and the length is always right there.

---

## String Slices (`&str`)

A string slice has the type `&str`. It's a reference to a portion of a `String` (or a string literal).

### Creating String Slices

```rust
fn main() {
    let s = String::from("hello world");
    
    // [start..end] — start is inclusive, end is exclusive
    let hello = &s[0..5];   // "hello" (bytes 0, 1, 2, 3, 4)
    let world = &s[6..11];  // "world" (bytes 6, 7, 8, 9, 10)
    let space = &s[5..6];   // " " (byte 5)
    
    println!("{} {} {}", hello, space, world);
}
```

### Range Syntax Shortcuts

```rust
let s = String::from("hello world");

// Full range:
let full = &s[0..11];   // "hello world"

// Start from 0:
let hello = &s[..5];    // Same as &s[0..5]

// Go to end:
let world = &s[6..];    // Same as &s[6..11]

// Entire string:
let all = &s[..];       // Same as &s[0..11]
```

### Byte Boundaries and UTF-8

**WARNING:** String slice indices are **byte** positions, not character positions. Rust strings are UTF-8, and some characters are more than 1 byte:

```rust
fn main() {
    let s = String::from("Здравствуйте");  // Russian, 2 bytes per char
    
    // let slice = &s[0..1];  // ❌ PANIC! Byte 1 is mid-character
    let slice = &s[0..2];     // ✅ "З" (first character, 2 bytes)
    println!("{}", slice);
}
```

```
ASCII characters:    1 byte  each (a, b, 1, @)
Cyrillic, etc.:      2 bytes each (З, д, р)
CJK characters:      3 bytes each (你, 好)
Emoji:               4 bytes each (😀, 🦀)
```

If you slice at a non-boundary, Rust panics at runtime:

```
thread 'main' panicked at 'byte index 1 is not a char boundary'
```

### String Literals Are Slices

Every string literal in Rust is already a `&str`:

```rust
let s: &str = "hello world";
```

```
Binary / Read-only Memory:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  ↑
  │
Stack:
┌────────────┐
│ ptr ───────┘
│ len: 11    │
└────────────┘
s: &str
```

String literals are stored in the binary itself. They live for the entire program duration (they have `'static` lifetime — we'll cover lifetimes later).

---

## Array and Vec Slices (`&[T]`)

Slices aren't just for strings. You can slice any contiguous collection.

### Creating Array Slices

```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];
    
    let slice: &[i32] = &arr[1..4];   // [2, 3, 4]
    let first_two = &arr[..2];         // [1, 2]
    let last_three = &arr[2..];        // [3, 4, 5]
    let all = &arr[..];               // [1, 2, 3, 4, 5]
    
    println!("{:?}", slice);       // [2, 3, 4]
    println!("{:?}", first_two);   // [1, 2]
    println!("{:?}", last_three);  // [3, 4, 5]
}
```

```
Stack:
arr:   [1, 2, 3, 4, 5]
        ↑  ↑        ↑
        │  │        │
        │  └─slice──┘ (ptr to index 1, len 3)
        │
        └─first_two (ptr to index 0, len 2)
```

### Vec Slices

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];
    
    let middle: &[i32] = &v[1..4];  // [20, 30, 40]
    let all: &[i32] = &v;           // Deref coercion: &Vec<i32> → &[i32]
    
    println!("{:?}", middle);  // [20, 30, 40]
    println!("{:?}", all);     // [10, 20, 30, 40, 50]
}
```

```
Stack:                    Heap:
v: ┌──────────┐          ┌────┬────┬────┬────┬────┐
   │ ptr ──────────────→ │ 10 │ 20 │ 30 │ 40 │ 50 │
   │ len: 5   │          └────┴────┴────┴────┴────┘
   │ cap: 5   │            ↑    ↑              ↑
   └──────────┘            │    │              │
                           │    └── middle ────┘
all: ┌────────┐            │    (ptr to [1], len 3)
     │ ptr ────────────────┘
     │ len: 5 │
     └────────┘
```

---

## Slices Enforce Borrowing Rules

Remember the problem from the beginning? The slice is tied to the original data:

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }
    
    s  // The whole string is one word
}

fn main() {
    let mut s = String::from("hello world");
    let word = first_word(&s);  // Immutable borrow via slice
    
    // s.clear();  // ❌ Cannot mutate while slice exists!
    //             // clear() takes &mut self, but we already have &str
    
    println!("First word: {}", word);
    
    s.clear();  // ✅ After word is done being used
}
```

The compiler catches the bug:

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:13:5
  |
11 |     let word = first_word(&s);
   |                           -- immutable borrow occurs here
12 |
13 |     s.clear();
   |     ^^^^^^^^^ mutable borrow occurs here
14 |     println!("First word: {}", word);
   |                                ---- immutable borrow later used here
```

---

## Mutable Slices (`&mut [T]`)

Just like references, slices can be mutable:

```rust
fn main() {
    let mut arr = [1, 2, 3, 4, 5];
    
    let slice = &mut arr[1..4];  // Mutable slice of [2, 3, 4]
    
    slice[0] = 20;  // Modify through slice
    slice[1] = 30;
    slice[2] = 40;
    
    println!("{:?}", arr);  // [1, 20, 30, 40, 5]
}
```

Mutable slices follow the same either/or rule:

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    
    // Can't have two mutable slices of the same data:
    // let a = &mut v[..3];
    // let b = &mut v[3..];  // ❌ Two mutable borrows of v
    
    // But split_at_mut is OK — it guarantees non-overlapping:
    let (a, b) = v.split_at_mut(3);
    a[0] = 10;
    b[0] = 40;
    
    println!("{:?}", v);  // [10, 2, 3, 40, 5, 6]
}
```

---

## Slices in Functions

### Why Accept `&[T]` Instead of `&Vec<T>`

```rust
// ❌ Too specific — only works with Vec<i32>
fn sum(numbers: &Vec<i32>) -> i32 {
    numbers.iter().sum()
}

// ✅ Generic — works with Vec, array, or any slice
fn sum(numbers: &[i32]) -> i32 {
    numbers.iter().sum()
}

fn main() {
    let v = vec![1, 2, 3];
    let arr = [4, 5, 6];
    
    println!("{}", sum(&v));        // Vec → &[i32] via deref coercion
    println!("{}", sum(&arr));      // Array → &[i32]
    println!("{}", sum(&v[1..]));   // Slice of Vec
    println!("{}", sum(&arr[..2])); // Slice of array
}
```

### Why Accept `&str` Instead of `&String`

```rust
// ❌ Too specific
fn greet(name: &String) {
    println!("Hello, {}!", name);
}

// ✅ Works with String, &str, and slices
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let owned = String::from("Alice");
    let literal = "Bob";
    
    greet(&owned);       // &String → &str via deref coercion
    greet(literal);      // &str directly
    greet(&owned[..3]);  // Slice "Ali"
}
```

**Rule of thumb:** In function parameters, prefer `&str` over `&String` and `&[T]` over `&Vec<T>`.

---

### Slices in Other Languages

Rust slices look simple — pointer + length — but comparing them to other languages reveals just how much Rust gets right. Not every language even *has* slices, and among those that do, the semantics vary wildly.

#### Go — The Closest Cousin

Go slices are the most similar to Rust. They're a built-in type with three fields:

```
Go slice header — 24 bytes:
┌──────────┐
│ pointer  │  → points to underlying array
├──────────┤
│ length   │  → number of elements currently visible
├──────────┤
│ capacity │  → total allocated space in the backing array
└──────────┘
```

Go slices carry **capacity** in addition to length, which means they can grow (via `append`) without the caller knowing the backing array changed. This is convenient but dangerous — two Go slices can alias the same array and silently corrupt each other's data. Rust's borrow checker makes this impossible.

#### Python — Slices Copy

In Python, slicing **always creates a new list**:

```python
my_list = [1, 2, 3, 4, 5]
slice = my_list[1:4]    # Allocates a NEW list [2, 3, 4]
slice[0] = 99           # Only modifies the copy
print(my_list)           # [1, 2, 3, 4, 5] — unchanged!
```

This is safe (no aliasing) but expensive for large data. Rust's slices are zero-copy views — no allocation, no copying, just a pointer and length.

#### JavaScript — It Depends

```javascript
// Array.prototype.slice() — COPIES
const arr = [1, 2, 3, 4, 5];
const s = arr.slice(1, 4);  // New array [2, 3, 4]

// TypedArray.subarray() — VIEW (like Rust slices)
const buf = new Int32Array([1, 2, 3, 4, 5]);
const view = buf.subarray(1, 4);  // View into same buffer
view[0] = 99;
console.log(buf);  // Int32Array [1, 99, 3, 4, 5]
```

JavaScript has no borrow checker, so `subarray` views can lead to surprising mutations.

#### C++ — `std::span` (C++20)

C++20 introduced `std::span<T>`, which is the closest equivalent to Rust's `&[T]`:

```cpp
#include <span>
void process(std::span<int> data) {
    for (int x : data) { /* ... */ }
}
```

Like Rust slices, `std::span` is a non-owning view (pointer + size). Unlike Rust, C++ has no borrow checker — you can easily create a dangling span.

#### C — Do It Yourself

C has no slice concept at all. You pass a pointer and a size as separate arguments:

```c
void process(int *data, size_t len) {
    for (size_t i = 0; i < len; i++) {
        printf("%d ", data[i]);
    }
}
```

Nothing stops you from passing the wrong length, reading past the end, or using the pointer after the data is freed. Every buffer overflow exploit in history is a consequence of this design.

#### Comparison Table

| Language | Slice Type | Copies Data? | Bounds Checked? | Lifetime Safe? |
|----------|-----------|-------------|----------------|---------------|
| **Rust** | `&[T]`, `&str` | No (view) | Yes (panic on OOB) | Yes (borrow checker) |
| **Go** | `[]T` | No (view) | Yes (panic on OOB) | No (GC, but aliasing bugs) |
| **Python** | `list[a:b]` | Yes (copy) | Yes (silent clamp) | N/A (GC) |
| **JavaScript** | `Array.slice()` | Yes (copy) | Yes (silent clamp) | N/A (GC) |
| **C++** | `std::span<T>` | No (view) | Optional (`.at()`) | No (dangling possible) |
| **C** | `ptr + len` | No (manual) | No | No |
| **Java** | `Arrays.copyOfRange()` | Yes (copy) | Yes (exception) | N/A (GC) |

Rust is the only language in this list where slices are zero-copy **and** lifetime-safe. The borrow checker proves at compile time that your slice won't outlive the data it points to — no garbage collector needed, no runtime cost, no surprises.

---

## Common Slice Methods

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7, 8];
    let s: &[i32] = &v;
    
    // Length
    println!("len: {}", s.len());           // 8
    println!("is_empty: {}", s.is_empty()); // false
    
    // First and last
    println!("first: {:?}", s.first());     // Some(1)
    println!("last: {:?}", s.last());       // Some(8)
    
    // Contains
    println!("contains 5: {}", s.contains(&5));  // true
    
    // Iteration
    for item in s.iter() {
        print!("{} ", item);
    }
    println!();
    
    // Windows: sliding window of size N
    for window in s.windows(3) {
        println!("window: {:?}", window);
    }
    // [1,2,3], [2,3,4], [3,4,5], [4,5,6], [5,6,7], [6,7,8]
    
    // Chunks: split into groups of size N
    for chunk in s.chunks(3) {
        println!("chunk: {:?}", chunk);
    }
    // [1,2,3], [4,5,6], [7,8]
    
    // Split
    let parts: Vec<&[i32]> = s.split(|&x| x == 4).collect();
    println!("split at 4: {:?}", parts);  // [[1,2,3], [5,6,7,8]]
}
```

String slice methods:

```rust
fn main() {
    let s = "Hello, World! Hello, Rust!";
    
    println!("starts_with: {}", s.starts_with("Hello"));  // true
    println!("ends_with: {}", s.ends_with("Rust!"));      // true
    println!("contains: {}", s.contains("World"));        // true
    
    // Find
    println!("find: {:?}", s.find("World"));  // Some(7)
    
    // Trim
    let padded = "  hello  ";
    println!("trim: '{}'", padded.trim());  // 'hello'
    
    // Split
    for word in s.split_whitespace() {
        print!("[{}] ", word);
    }
    println!();
    // [Hello,] [World!] [Hello,] [Rust!]
}
```

---

## Slice Patterns

### Pattern 1: Find-in-Slice

```rust
fn find_max(slice: &[i32]) -> Option<&i32> {
    if slice.is_empty() {
        return None;
    }
    let mut max = &slice[0];
    for item in &slice[1..] {
        if item > max {
            max = item;
        }
    }
    Some(max)
}

fn main() {
    let data = vec![3, 7, 2, 9, 4];
    if let Some(max) = find_max(&data) {
        println!("Max: {}", max);  // 9
    }
}
```

### Pattern 2: Split and Process

```rust
fn parse_csv_line(line: &str) -> Vec<&str> {
    line.split(',')
        .map(|field| field.trim())
        .collect()
}

fn main() {
    let line = "Alice, 30, Engineer";
    let fields = parse_csv_line(line);
    println!("{:?}", fields);  // ["Alice", "30", "Engineer"]
    
    // Each field is a &str pointing into `line` — no allocation!
}
```

### Pattern 3: Windows and Chunks

```rust
// Find pairs of consecutive equal elements
fn find_consecutive_duplicates(data: &[i32]) -> Vec<i32> {
    let mut result = Vec::new();
    for window in data.windows(2) {
        if window[0] == window[1] {
            result.push(window[0]);
        }
    }
    result
}

// Process data in batches
fn process_in_batches(data: &[i32], batch_size: usize) {
    for (i, chunk) in data.chunks(batch_size).enumerate() {
        let sum: i32 = chunk.iter().sum();
        println!("Batch {}: {:?} (sum: {})", i, chunk, sum);
    }
}

fn main() {
    let data = [1, 1, 2, 3, 3, 3, 4];
    let dups = find_consecutive_duplicates(&data);
    println!("Consecutive duplicates: {:?}", dups);  // [1, 3, 3]
    
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    process_in_batches(&numbers, 3);
    // Batch 0: [1, 2, 3] (sum: 6)
    // Batch 1: [4, 5, 6] (sum: 15)
    // Batch 2: [7, 8, 9] (sum: 24)
    // Batch 3: [10] (sum: 10)
}
```

### Pattern 4: Slice of Slices

```rust
fn main() {
    let words = ["hello", "world", "foo", "bar"];
    
    // words is [&str; 4]
    // &words[..] is &[&str] — a slice of string slices!
    let first_two: &[&str] = &words[..2];
    println!("{:?}", first_two);  // ["hello", "world"]
}
```

---

## How Slices Prevent Bugs

### Bug 1: Out-of-sync indices (solved)

Without slices (indices can become stale):
```rust
let end = first_word_index(&s);  // Returns 5
s.clear();                       // s is now empty
// end is still 5 — meaningless!
```

With slices (compiler catches it):
```rust
let word = first_word(&s);  // Returns &str slice
// s.clear();               // ❌ Compiler error! Can't modify while slice exists
println!("{}", word);
```

### Bug 2: Out-of-bounds access

```rust
let v = vec![1, 2, 3];
// let s = &v[0..10];  // Panics at runtime: index out of bounds
let s = v.get(0..10);  // Returns None safely
```

### Bug 3: Dangling references

Slices are references, so the borrow checker prevents dangling:
```rust
// fn bad() -> &str {
//     let s = String::from("hello");
//     &s[..]  // ❌ s is dropped, slice would dangle
// }
```

---

## Common Mistakes

### Mistake 1: Slicing Multi-Byte Characters

```rust
let s = String::from("café");
// 'é' is 2 bytes in UTF-8
// let slice = &s[0..4];  // Might panic depending on encoding!

// Safe approach: use char_indices
for (i, c) in s.char_indices() {
    println!("byte {}: {}", i, c);
}
// byte 0: c
// byte 1: a
// byte 2: f
// byte 3: é  (bytes 3-4)
```

### Mistake 2: Assuming Slice Indices Are Character Indices

```rust
let s = "Hello 🌍";
println!("Bytes: {}", s.len());    // 10 (🌍 is 4 bytes)
println!("Chars: {}", s.chars().count());  // 7

// &s[0..7] is NOT the first 7 characters!
// It's the first 7 bytes: "Hello 🌍" would need &s[0..10]
```

### Mistake 3: Thinking Slices Copy Data

```rust
let v = vec![1, 2, 3, 4, 5];
let slice = &v[1..4];  // This does NOT copy [2, 3, 4]
                         // It's just a pointer + length
```

---

### UTF-8 and String Slices — Why You Can't Index Strings

If you've come from Python, JavaScript, or Java, you might be frustrated that Rust won't let you write `s[2]` on a `String`. This isn't a limitation — it's Rust protecting you from a class of bugs that plagues other languages. To understand why, we need to talk about text encoding.

#### A Brief History of Text Encoding

```
1963  ASCII           7 bits    128 characters   (English letters, digits, symbols)
1981  Extended ASCII  8 bits    256 characters   (added accented chars, box-drawing)
1991  Unicode 1.0     16 bits   7,161 characters (ambitious but underestimated)
2024  Unicode 16.0    21 bits   149,813 characters (every script, emoji, historic symbols)
```

Unicode assigns a unique **code point** (a number like U+0041 for 'A' or U+1F980 for 🦀) to every character. But how do you store these numbers in memory?

#### The Encoding Problem

| Encoding | Bytes per char | Pros | Cons |
|----------|---------------|------|------|
| **UTF-32** | Always 4 | Simple indexing: `s[i]` just works | Wastes 3 bytes for every ASCII char |
| **UTF-16** | 2 or 4 | Compromise, used by Java/JS/Windows | Surrogate pairs are confusing; still wastes space for ASCII |
| **UTF-8** | 1, 2, 3, or 4 | Compact for ASCII, backward compatible | Variable width — can't index by character |

UTF-8 was designed in **1992** by Ken Thompson and Rob Pike — reportedly sketched out on a placemat in a New Jersey diner. Its design is elegant:

```
Bytes   Bits   Range                  Example
1 byte   7     U+0000  – U+007F       'A' = 0x41
2 bytes  11    U+0080  – U+07FF       'é' = 0xC3 0xA9
3 bytes  16    U+0800  – U+FFFF       '你' = 0xE4 0xBD 0xA0
4 bytes  21    U+10000 – U+10FFFF     '🦀' = 0xF0 0x9F 0xA6 0x80
```

UTF-8 won the encoding war decisively: over 98% of websites use it. It's backward compatible with ASCII (every valid ASCII file is also valid UTF-8), it's self-synchronizing (you can jump into the middle of a stream and find character boundaries), and it has no byte-order issues.

#### Why Rust Chose UTF-8

Rust's `String` and `&str` are **always** valid UTF-8. This was a deliberate choice:
- It's the web's encoding — if you're parsing HTTP, JSON, HTML, or TOML, your data is almost certainly UTF-8
- It's the most space-efficient encoding for mixed content (English text with occasional emoji)
- It's what Linux, macOS, and modern tools use natively

#### The Indexing Problem

Consider the string `"café"`:

```
Character:  c     a     f     é
Bytes:      0x63  0x61  0x66  0xC3 0xA9
Byte index: [0]   [1]   [2]   [3]  [4]
Char index: [0]   [1]   [2]   [3]
```

The string is **5 bytes** but only **4 characters**. Now, what should `s[3]` return?

- **Byte 3?** That's `0xC3` — the first half of 'é'. Not a valid character!
- **Character 3?** That's 'é' — but finding it requires scanning from the start (O(n)).
- **Grapheme cluster 3?** Usually the same as character 3, but not always (think combining accents).

Rust refuses to guess. Instead, it forces you to be explicit:

```rust
fn main() {
    let s = String::from("café");

    // Byte-level access
    for b in s.bytes() {
        print!("{:#04x} ", b);  // 0x63 0x61 0x66 0xc3 0xa9
    }
    println!();

    // Character (Unicode scalar) access
    for c in s.chars() {
        print!("'{}' ", c);  // 'c' 'a' 'f' 'é'
    }
    println!();

    // Safe slicing via char_indices
    for (byte_pos, ch) in s.char_indices() {
        println!("byte {}: '{}'", byte_pos, ch);
    }
    // byte 0: 'c'
    // byte 1: 'a'
    // byte 2: 'f'
    // byte 3: 'é'
}
```

For grapheme clusters (user-perceived characters like 👨‍👩‍👧‍👦, which is **one** grapheme but **seven** Unicode scalars and **25 bytes**), use the `unicode-segmentation` crate.

#### This Is Not a Limitation — It's a Feature

Java and JavaScript use UTF-16 internally and expose `charAt(i)` — which silently breaks on emoji:

```javascript
// JavaScript
"😀".length          // 2 (!!) — it's a surrogate pair
"😀"[0]              // "\uD83D" — half a smiley face
"😀".charAt(0)       // "\uD83D" — still broken
```

Python 3 hides the encoding behind an abstraction, so `s[i]` works on characters — but at the cost of storing every string in the widest encoding needed (up to 4 bytes per char internally).

Rust's approach is honest: UTF-8 strings have variable-width characters, so byte indexing is the only O(1) operation. Rather than pretending otherwise and introducing subtle bugs, Rust makes you choose your iteration strategy explicitly. Every Rust program that compiles handles Unicode correctly by construction.

---

## Exercises

### Exercise 1: Second Word

Write a function that returns the second word of a string (or the empty string if there's no second word):

```rust
fn second_word(s: &str) -> &str {
    // Your code here
}

fn main() {
    assert_eq!(second_word("hello world"), "world");
    assert_eq!(second_word("hello"), "");
    assert_eq!(second_word("one two three"), "two");
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn second_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    let mut first_space = None;
    
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            if first_space.is_none() {
                first_space = Some(i);
            } else {
                // Found second space
                return &s[first_space.unwrap() + 1..i];
            }
        }
    }
    
    // If we found one space but no second, return from after the space
    match first_space {
        Some(i) => &s[i + 1..],
        None => "",
    }
}
```

</details>

### Exercise 2: Sum of Middle Elements

Write a function that takes a slice and returns the sum of all elements except the first and last:

```rust
fn sum_middle(slice: &[i32]) -> i32 {
    // Your code here
}

fn main() {
    assert_eq!(sum_middle(&[1, 2, 3, 4, 5]), 9);  // 2 + 3 + 4
    assert_eq!(sum_middle(&[10, 20]), 0);           // No middle elements
    assert_eq!(sum_middle(&[5]), 0);                // No middle elements
    assert_eq!(sum_middle(&[]), 0);                 // Empty
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn sum_middle(slice: &[i32]) -> i32 {
    if slice.len() <= 2 {
        return 0;
    }
    slice[1..slice.len() - 1].iter().sum()
}
```

</details>

### Exercise 3: Will It Compile?

**A:**
```rust
fn main() {
    let mut s = String::from("hello world");
    let word = &s[..5];
    s.push_str("!");
    println!("{}", word);
}
```

**B:**
```rust
fn main() {
    let mut s = String::from("hello world");
    let word = &s[..5];
    println!("{}", word);
    s.push_str("!");
}
```

<details>
<summary>Answers</summary>

- **A:** ❌ Does not compile. `word` (a `&str` slice) borrows `s` immutably, then `push_str` tries to borrow mutably while `word` is still used afterward.
- **B:** ✅ Compiles. `word` is last used in `println!`, and then `push_str` occurs after the immutable borrow has ended (NLL).

</details>

### Exercise 4: Longest Common Prefix

Write a function that finds the longest common prefix of two string slices:

```rust
fn longest_common_prefix<'a>(a: &'a str, b: &str) -> &'a str {
    // Your code here
}

fn main() {
    assert_eq!(longest_common_prefix("hello", "help"), "hel");
    assert_eq!(longest_common_prefix("rust", "ruby"), "ru");
    assert_eq!(longest_common_prefix("abc", "xyz"), "");
    assert_eq!(longest_common_prefix("same", "same"), "same");
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn longest_common_prefix<'a>(a: &'a str, b: &str) -> &'a str {
    let mut end = 0;
    for (ca, cb) in a.chars().zip(b.chars()) {
        if ca != cb {
            break;
        }
        end += ca.len_utf8();
    }
    &a[..end]
}
```

</details>

---

## Summary

### Slice Types at a Glance

| Type | What it slices | Size on stack |
|------|----------------|---------------|
| `&str` | `String` or string literal | 16 bytes (ptr + len) |
| `&[T]` | `Vec<T>` or `[T; N]` | 16 bytes (ptr + len) |
| `&mut str` | Mutable string data | 16 bytes (ptr + len) |
| `&mut [T]` | Mutable array/vec data | 16 bytes (ptr + len) |

### Range Syntax

| Syntax | Meaning |
|--------|---------|
| `&s[0..5]` | Bytes/elements 0 through 4 |
| `&s[..5]` | From start through 4 |
| `&s[2..]` | From 2 to end |
| `&s[..]` | The entire collection |
| `&s[2..=5]` | Bytes/elements 2 through 5 (inclusive) |

### Key Takeaways

1. **Slices are fat pointers** — pointer + length, 16 bytes total
2. **No data copying** — slices reference existing data
3. **Slices follow borrowing rules** — the compiler ensures data stays valid
4. **String slices index bytes, not characters** — be careful with UTF-8
5. **String literals are `&str`** — they live in the binary
6. **Prefer `&str` over `&String` in function params** — more flexible
7. **Prefer `&[T]` over `&Vec<T>` in function params** — more flexible
8. **Slices connect references to data** — preventing stale index bugs

---

## What's Next?

Now that you understand slices and `&str`, the next tutorial dives deep into one of the most common confusions for Rust beginners: **`String` vs `&str`** — when to use which and how they relate.

**Next Tutorial:** [String vs &str →](./08-string-vs-str.md)

---

<p align="center">
  <i>Tutorial 7 of 8 — Stage 3: Ownership</i>
</p>

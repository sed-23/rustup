# Vectors — Vec<T> 📋

> **`Vec<T>` is Rust's growable array — the most commonly used collection. It stores elements of the same type in a contiguous block of heap memory that can resize dynamically.**

---

## Table of Contents

- [What Is a Vector?](#what-is-a-vector)
- [Creating Vectors](#creating-vectors)
- [Reading Elements](#reading-elements)
- [Modifying Vectors](#modifying-vectors)
- [Iterating](#iterating)
- [Vectors and Ownership](#vectors-and-ownership)
- [Common Methods](#common-methods)
- [Storing Multiple Types](#storing-multiple-types)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is a Vector?

A `Vec<T>` is a dynamic array — like `ArrayList` in Java or `list` in Python. It:
- Stores elements of type `T` in contiguous heap memory
- Grows and shrinks as needed
- Provides O(1) indexing and O(1) amortized push to the end

---

### Dynamic Arrays Across Programming Languages

Rust's `Vec<T>` is not a new idea — virtually every language has a growable array. But they all make different tradeoffs:

| Language | Type | Contiguous? | Typed? | Growth Strategy | Memory Managed By |
|----------|------|-------------|--------|-----------------|-------------------|
| C | `malloc`/`realloc` | Yes | Manual casting | Manual | Programmer (`free`) |
| C++ | `std::vector<T>` | Yes | Yes (templates) | 2x (typically) | RAII (destructor) |
| Java | `ArrayList<T>` | Yes* | Objects only† | 1.5x | Garbage collector |
| Python | `list` | Yes* | No (any type) | ~1.125x | Reference counting + GC |
| JavaScript | `Array` | Sometimes‡ | No (any type) | Engine-dependent | Garbage collector |
| Go | `[]T` (slice) | Yes | Yes | 2x (small), 1.25x (large) | Garbage collector |
| **Rust** | **`Vec<T>`** | **Yes** | **Yes (generics)** | **2x** | **Ownership system** |

> \* Java's `ArrayList` and Python's `list` store *references* contiguously, but the objects themselves may be scattered in heap memory.  
> † Primitives like `int` require boxing to `Integer` in Java (before Project Valhalla).  
> ‡ V8 optimizes dense arrays into true contiguous storage, but sparse arrays fall back to hash-map-like structures.

**In C**, there is no built-in dynamic array. You call `malloc()` to allocate, `realloc()` when you need more space, and `free()` when you're done. You track the length and capacity yourself — and a single mistake means memory corruption or a leak:

```c
// C: manual dynamic array (error-prone)
int *arr = malloc(4 * sizeof(int));
int len = 0, cap = 4;
arr[len++] = 10;
arr[len++] = 20;
if (len == cap) {
    cap *= 2;
    arr = realloc(arr, cap * sizeof(int));  // hope this doesn't fail...
}
free(arr);  // forget this and you leak memory
```

**In C++**, `std::vector<T>` is essentially the same data structure as Rust's `Vec<T>` — contiguous memory, automatic reallocation, templated over the element type. Rust borrowed the *name* "vector" directly from C++.

> **Why "vector"?** The name has nothing to do with mathematical vectors (direction + magnitude). The C++ standard library chose it early on, and the name stuck across languages. Bjarne Stroustrup has said he would have preferred "array" if it weren't already taken.

**The fundamental insight:** all of these are the *same data structure* — a contiguous block of memory with a length and capacity, that reallocates to a larger block when full. The differences are in type safety, memory management, and what guarantees the language provides. Rust's `Vec<T>` gives you C++-level performance with compile-time memory safety — no garbage collector, no manual `free()`.

---

## Creating Vectors

```rust
fn main() {
    // Empty vector (type annotation needed)
    let v: Vec<i32> = Vec::new();
    
    // With the vec! macro (type inferred)
    let v = vec![1, 2, 3, 4, 5];
    
    // Repeating value
    let zeros = vec![0; 10];  // Ten zeros: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    
    // With capacity (pre-allocates, avoids resizing)
    let mut v = Vec::with_capacity(100);
    v.push(1);  // No reallocation for first 100 pushes
    
    // From an iterator
    let v: Vec<i32> = (1..=5).collect();  // [1, 2, 3, 4, 5]
    
    // From an array
    let arr = [1, 2, 3];
    let v = arr.to_vec();
}
```

---

## Reading Elements

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];
    
    // Index (panics if out of bounds)
    let third = v[2];
    println!("Third: {}", third);  // 30
    // let oob = v[10];  // PANIC: index out of bounds
    
    // .get() returns Option (safe)
    match v.get(2) {
        Some(val) => println!("Third: {}", val),
        None => println!("No third element"),
    }
    
    // First and last
    println!("First: {:?}", v.first());  // Some(10)
    println!("Last: {:?}", v.last());    // Some(50)
    
    // Length
    println!("Length: {}", v.len());     // 5
    println!("Empty: {}", v.is_empty()); // false
}
```

---

## Modifying Vectors

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    
    // Add to end
    v.push(4);            // [1, 2, 3, 4]
    
    // Remove from end
    let last = v.pop();   // Some(4), v = [1, 2, 3]
    
    // Insert at index (shifts elements right)
    v.insert(1, 10);      // [1, 10, 2, 3]
    
    // Remove at index (shifts elements left)
    v.remove(1);           // [1, 2, 3]
    
    // Extend with another iterator
    v.extend([4, 5, 6]);  // [1, 2, 3, 4, 5, 6]
    
    // Retain only elements matching predicate
    v.retain(|&x| x % 2 == 0);  // [2, 4, 6]
    
    // Sort
    let mut nums = vec![3, 1, 4, 1, 5, 9];
    nums.sort();           // [1, 1, 3, 4, 5, 9]
    nums.dedup();          // [1, 3, 4, 5, 9] (remove consecutive dupes)
    
    // Reverse
    nums.reverse();        // [9, 5, 4, 3, 1]
    
    // Truncate
    nums.truncate(3);      // [9, 5, 4]
    
    // Clear
    nums.clear();          // []
}
```

---

## Iterating

```rust
fn main() {
    let v = vec![10, 20, 30];
    
    // Immutable iteration
    for val in &v {
        println!("{}", val);
    }
    
    // Mutable iteration
    let mut v = vec![1, 2, 3];
    for val in &mut v {
        *val *= 10;
    }
    println!("{:?}", v);  // [10, 20, 30]
    
    // Consuming iteration (moves values out)
    let v = vec![String::from("a"), String::from("b")];
    for s in v {
        println!("{}", s);
    }
    // v is no longer usable — ownership moved
    
    // With index
    for (i, val) in v.iter().enumerate() {
        println!("[{}] = {}", i, val);
    }
}
```

---

## Vectors and Ownership

```rust
fn main() {
    let mut v = vec![String::from("hello"), String::from("world")];
    
    // Borrowing an element
    let first = &v[0];  // Immutable borrow
    println!("{}", first);
    // v.push(String::from("!")); // ❌ Can't push while first is borrowed
    //                              // push might reallocate, invalidating first
    
    // After first is done:
    v.push(String::from("!"));  // ✅ OK now
    
    // Moving out of a Vec
    // let s = v[0];  // ❌ Can't move out of indexed content
    let s = v.remove(0);  // ✅ Removes and returns
    let s = v.swap_remove(0);  // ✅ Swaps with last, removes (O(1))
}
```

---

## Common Methods

```rust
fn main() {
    let v = vec![3, 1, 4, 1, 5, 9, 2, 6];
    
    println!("contains 5: {}", v.contains(&5));  // true
    println!("position of 4: {:?}", v.iter().position(|&x| x == 4));  // Some(2)
    
    // Slicing
    let slice: &[i32] = &v[2..5];  // [4, 1, 5]
    
    // Splitting
    let (left, right) = v.split_at(4);
    println!("{:?} {:?}", left, right);  // [3, 1, 4, 1] [5, 9, 2, 6]
    
    // Windows and chunks
    for window in v.windows(3) {
        print!("{:?} ", window);
    }
    println!();
    
    for chunk in v.chunks(3) {
        print!("{:?} ", chunk);
    }
    println!();
    
    // Joining (for Vec<String> or Vec<&str>)
    let words = vec!["hello", "world"];
    println!("{}", words.join(", "));  // "hello, world"
}
```

---

### Vectors and the CPU Cache — Why Contiguous Memory Matters

Why is `Vec<T>` the default collection in Rust? The answer lives in your CPU's cache.

**Cache lines:** Modern CPUs don't fetch memory one byte at a time. They load **cache lines** — typically 64 bytes. When you access one element of a `Vec<i32>`, the CPU loads 64 bytes around it, giving you **16 `i32` values for free**:

```
  Cache line (64 bytes)
  ┌──────────────────────────────────────────────────────────────┐
  │ i32 │ i32 │ i32 │ i32 │ i32 │ i32 │ i32 │ i32 │ ... (×16) │
  └──────────────────────────────────────────────────────────────┘
  Vec<i32>:  [  1,   2,   3,   4,   5,   6,   7,   8, ... ]
               ↑ access [0] → CPU loads [0] through [15] into cache
```

When you iterate sequentially, the **prefetcher** detects the pattern and loads the *next* cache line before you even ask for it. This means iterating a `Vec` is essentially limited only by memory bandwidth — not latency.

**Compare with `LinkedList`:**

```
  LinkedList nodes scattered in heap:
  ┌───────┐     ┌───────┐     ┌───────┐
  │ val:1 │──→  │ val:2 │──→  │ val:3 │──→  ...
  │@0x1A00│     │@0x5F28│     │@0x3B10│
  └───────┘     └───────┘     └───────┘
       ↑              ↑              ↑
   cache miss!    cache miss!    cache miss!
```

Each node could be *anywhere* in memory. The CPU can't predict where the next node is, so every access is potentially a **cache miss** — meaning it must go all the way to main memory (~100 nanoseconds) instead of L1 cache (~1 nanosecond). That's a **100x** penalty per access.

**Real-world impact:**

| Operation | `Vec<i32>` (1M elements) | `LinkedList<i32>` (1M elements) |
|---|---|---|
| Sequential iteration | ~0.5 ms | ~15–30 ms |
| Sum all elements | ~0.3 ms | ~20 ms |
| Random access by index | O(1) | O(n) — must walk the list |

This is why Rust documentation recommends `Vec` as the go-to collection. Even when an algorithm *theoretically* favors a linked list (e.g., frequent insertion at the front), the cache advantage of `Vec` often wins in practice.

**Data-oriented design:** this principle — structuring your data for the cache, not for an object hierarchy — is at the core of high-performance software. Game engines like **Bevy** (written in Rust) store entity data in flat, contiguous `Vec` arrays organized by component type (an ECS architecture), rather than in object trees. The result is dramatically better iteration performance.

> **Rule of thumb:** always reach for `Vec` first. Only switch to another collection (`VecDeque`, `HashMap`, `BTreeMap`, etc.) when you have a *specific, measured* reason to do so.

---

## Storing Multiple Types

Vectors hold one type, but you can use enums:

```rust
#[derive(Debug)]
enum Cell {
    Int(i64),
    Float(f64),
    Text(String),
    Bool(bool),
}

fn main() {
    let row: Vec<Cell> = vec![
        Cell::Int(1),
        Cell::Text(String::from("Alice")),
        Cell::Float(3.14),
        Cell::Bool(true),
    ];
    
    for cell in &row {
        match cell {
            Cell::Int(n) => println!("Integer: {}", n),
            Cell::Float(f) => println!("Float: {:.2}", f),
            Cell::Text(s) => println!("Text: {}", s),
            Cell::Bool(b) => println!("Bool: {}", b),
        }
    }
}
```

---

### Vectors and Ownership — Common Patterns and Pitfalls

The borrow checker's interaction with `Vec` is where many Rust beginners first hit a wall. Let's break down the most common patterns.

**The classic error: borrowing + mutating**

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    let first = &v[0];  // immutable borrow of v
    v.push(6);           // ❌ mutable borrow of v
    println!("{}", first);
}
```

```
error[E0502]: cannot borrow `v` as mutable because it is
              also borrowed as immutable
```

**Why does Rust prevent this?** When you `push`, the `Vec` *might* need to reallocate to a larger buffer. If it does, all existing references point to freed memory — a **use-after-free** bug. This is exactly **iterator invalidation** in C++, which is a real source of production bugs. Rust catches it at compile time.

```
  Before push:                After reallocation:
  ┌───┬───┬───┬───┬───┐      ┌───┬───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 4 │ 5 │      │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │  (new buffer)
  └───┴───┴───┴───┴───┘      └───┴───┴───┴───┴───┴───┘
    ↑ first points here         (old buffer is freed!)
    ⚠ dangling pointer!         first → 💀 freed memory
```

**Solutions to the borrow conflict:**

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];

    // Solution 1: limit the scope of the borrow
    {
        let first = &v[0];
        println!("{}", first);
    } // first is dropped here
    v.push(6); // ✅ no active borrows

    // Solution 2: use indices instead of references
    let first_index = 0;
    v.push(7);
    println!("{}", v[first_index]); // ✅ index is just a number

    // Solution 3: clone the value
    let first = v[0]; // Copy for i32 (or .clone() for String)
    v.push(8);
    println!("{}", first); // ✅ it's an independent copy
}
```

**The `drain()` pattern** — efficiently remove a range of elements:

```rust
fn main() {
    let mut v = vec![10, 20, 30, 40, 50];
    let removed: Vec<i32> = v.drain(1..3).collect();  // remove indices 1, 2
    println!("removed: {:?}", removed);  // [20, 30]
    println!("remaining: {:?}", v);      // [10, 40, 50]
}
```

**The `retain()` pattern** — filter in place without allocating a new `Vec`:

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6, 7, 8];
    v.retain(|&x| x % 2 == 0);  // keep only evens
    println!("{:?}", v);  // [2, 4, 6, 8]
}
```

**Split borrowing with `split_at_mut()`** — borrow different parts mutably:

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    // You CAN'T do: let a = &mut v[0]; let b = &mut v[4]; (compiler can't prove no overlap)
    // But you CAN split:
    let (left, right) = v.split_at_mut(3);
    left[0] = 100;   // mutate index 0
    right[1] = 500;  // mutate index 4
    println!("{:?}", v);  // [100, 2, 3, 4, 500]
}
```

The compiler knows `left` and `right` don't overlap, so both mutable borrows are safe. This pattern is essential when you need to modify two parts of a `Vec` simultaneously.

---

## Exercises

### Exercise 1: Statistics

Write a function that takes `&[i32]` and returns the mean, median, and mode:

```rust
fn stats(data: &[i32]) -> (f64, f64, i32) {
    // Return (mean, median, mode)
}
```

<details>
<summary>Solution</summary>

```rust
use std::collections::HashMap;

fn stats(data: &[i32]) -> (f64, f64, i32) {
    let mean = data.iter().sum::<i32>() as f64 / data.len() as f64;
    
    let mut sorted = data.to_vec();
    sorted.sort();
    let median = if sorted.len() % 2 == 0 {
        (sorted[sorted.len()/2 - 1] + sorted[sorted.len()/2]) as f64 / 2.0
    } else {
        sorted[sorted.len()/2] as f64
    };
    
    let mut counts = HashMap::new();
    for &n in data {
        *counts.entry(n).or_insert(0) += 1;
    }
    let mode = *counts.iter().max_by_key(|&(_, count)| count).unwrap().0;
    
    (mean, median, mode)
}
```

</details>

---

## Summary

| Operation | Method | Time |
|-----------|--------|------|
| Push to end | `push(val)` | O(1) amortized |
| Pop from end | `pop()` | O(1) |
| Index | `v[i]` | O(1) |
| Safe index | `v.get(i)` | O(1) |
| Insert at i | `insert(i, val)` | O(n) |
| Remove at i | `remove(i)` | O(n) |
| Swap remove | `swap_remove(i)` | O(1) |
| Search | `contains(&val)` | O(n) |
| Sort | `sort()` | O(n log n) |
| Length | `len()` | O(1) |

---

**Next:** [Vector Memory Layout →](./02-vector-memory-layout.md)

<p align="center"><i>Tutorial 1 of 8 — Stage 5: Collections</i></p>

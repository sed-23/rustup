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

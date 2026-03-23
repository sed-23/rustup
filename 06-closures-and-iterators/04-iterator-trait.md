# The Iterator Trait 🔄

> **Iterators are Rust's way of processing sequences of values lazily. The `Iterator` trait has one required method — `next()` — and provides dozens of powerful built-in methods for transforming, filtering, and consuming data.**

---

## Table of Contents

- [What Is an Iterator?](#what-is-an-iterator)
- [The Iterator Trait](#the-iterator-trait)
- [Creating Iterators](#creating-iterators)
- [iter, iter_mut, into_iter](#iter-iter_mut-into_iter)
- [Laziness](#laziness)
- [for Loops and Iterators](#for-loops-and-iterators)
- [IntoIterator Trait](#intoiterator-trait)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is an Iterator?

An iterator is anything that produces a sequence of values, one at a time:

```rust
fn main() {
    let v = vec![10, 20, 30];
    
    // Create an iterator
    let mut iter = v.iter();
    
    // Call next() to get each value
    println!("{:?}", iter.next());  // Some(&10)
    println!("{:?}", iter.next());  // Some(&20)
    println!("{:?}", iter.next());  // Some(&30)
    println!("{:?}", iter.next());  // None (exhausted)
    println!("{:?}", iter.next());  // None (stays None forever)
}
```

---

## The Iterator Trait

```rust
pub trait Iterator {
    type Item;  // The type of each element
    
    fn next(&mut self) -> Option<Self::Item>;
    
    // ... 70+ provided methods (map, filter, fold, etc.)
}
```

Only `next()` is required. Everything else is built on top of it.

The `Item` associated type tells you what the iterator yields:

| Iterator | `Item` type |
|----------|-------------|
| `vec.iter()` | `&T` |
| `vec.iter_mut()` | `&mut T` |
| `vec.into_iter()` | `T` |
| `"hello".chars()` | `char` |
| `"hello".bytes()` | `u8` |
| `(0..10)` | `i32` |
| `hashmap.iter()` | `(&K, &V)` |

---

## Creating Iterators

```rust
fn main() {
    // From ranges
    let r = 0..5;           // 0, 1, 2, 3, 4
    let r = 0..=5;          // 0, 1, 2, 3, 4, 5
    let r = (0..10).step_by(2);  // 0, 2, 4, 6, 8
    
    // From collections
    let v = vec![1, 2, 3];
    let iter = v.iter();
    
    // From functions
    let ones = std::iter::repeat(1);           // 1, 1, 1, 1, ...
    let empty: std::iter::Empty<i32> = std::iter::empty();  // nothing
    let single = std::iter::once(42);          // just 42
    
    // From a closure (generate values)
    let mut count = 0;
    let counter = std::iter::from_fn(move || {
        count += 1;
        if count <= 5 { Some(count) } else { None }
    });
    let v: Vec<i32> = counter.collect();
    println!("{:?}", v);  // [1, 2, 3, 4, 5]
    
    // Successors (each value computed from previous)
    let powers: Vec<i32> = std::iter::successors(Some(1), |&prev| {
        let next = prev * 2;
        if next <= 128 { Some(next) } else { None }
    }).collect();
    println!("{:?}", powers);  // [1, 2, 4, 8, 16, 32, 64, 128]
}
```

---

## iter, iter_mut, into_iter

Every Rust collection provides three iterator methods:

### `.iter()` — borrows immutably

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    for val in v.iter() {
        // val is &i32
        println!("{}", val);
    }
    
    println!("{:?}", v);  // ✅ v still usable
}
```

### `.iter_mut()` — borrows mutably

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    
    for val in v.iter_mut() {
        // val is &mut i32
        *val *= 10;
    }
    
    println!("{:?}", v);  // [10, 20, 30]
}
```

### `.into_iter()` — takes ownership

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];
    
    for val in v.into_iter() {
        // val is String (owned)
        println!("{}", val);
    }
    
    // println!("{:?}", v);  // ❌ v was consumed
}
```

### Comparison Table

```
Method        | Item type | Ownership
--------------+-----------+------------------
.iter()       | &T        | Borrows collection
.iter_mut()   | &mut T    | Mutably borrows
.into_iter()  | T         | Consumes collection
```

---

## Laziness

**Iterator adaptors are lazy** — they do nothing until consumed:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // This does NOTHING yet — just builds a pipeline
    let pipeline = v.iter()
        .map(|x| {
            println!("mapping {}", x);  // This won't print!
            x * 2
        })
        .filter(|x| {
            println!("filtering {}", x);  // This won't print either!
            x > &4
        });
    
    println!("Pipeline created, nothing computed yet!");
    
    // NOW it runs (collect is a consumer)
    let result: Vec<&i32> = pipeline.collect();
    // mapping 1
    // filtering 2
    // mapping 2
    // filtering 4
    // mapping 3
    // filtering 6   ← kept
    // mapping 4
    // filtering 8   ← kept
    // mapping 5
    // filtering 10  ← kept
    
    // Note: items are processed ONE AT A TIME through the whole chain
    // NOT: map all, then filter all
}
```

Benefits of laziness:
- No intermediate collections allocated
- Can short-circuit (e.g., `find`, `any`, `take`)
- Can work with infinite iterators

```rust
fn main() {
    // Infinite iterator — only takes what's needed
    let first_10_squares: Vec<i32> = (0..)
        .map(|x| x * x)
        .take(10)
        .collect();
    
    println!("{:?}", first_10_squares);
    // [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
}
```

---

## for Loops and Iterators

`for` loops automatically call `.into_iter()`:

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    // These are equivalent:
    for x in v.iter() { /* x: &i32 */ }
    for x in &v { /* x: &i32 — &v calls v.iter() */ }
    
    let mut v = vec![1, 2, 3];
    for x in v.iter_mut() { /* x: &mut i32 */ }
    for x in &mut v { /* x: &mut i32 */ }
    
    let v = vec![1, 2, 3];
    for x in v.into_iter() { /* x: i32, v consumed */ }
    // for x in v { /* same thing */ }
}
```

Under the hood, a `for` loop desugars to:

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    // This:
    for x in &v {
        println!("{}", x);
    }
    
    // Is equivalent to:
    let mut iter = (&v).into_iter();
    loop {
        match iter.next() {
            Some(x) => println!("{}", x),
            None => break,
        }
    }
}
```

---

## IntoIterator Trait

`IntoIterator` converts something INTO an iterator:

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;
    
    fn into_iter(self) -> Self::IntoIter;
}
```

This is what `for` loops use. It's implemented for:
- `Vec<T>` → yields owned `T`
- `&Vec<T>` → yields `&T`
- `&mut Vec<T>` → yields `&mut T`
- Arrays, slices, HashMap, HashSet, BTreeMap, etc.

You can accept any iterable in your functions:

```rust
fn sum_all<I: IntoIterator<Item = i32>>(values: I) -> i32 {
    values.into_iter().sum()
}

fn main() {
    println!("{}", sum_all(vec![1, 2, 3]));     // 6
    println!("{}", sum_all([1, 2, 3]));          // 6
    println!("{}", sum_all(1..=3));              // 6
}
```

---

## Exercises

### Exercise 1: Manual Iterator

Manually call `next()` on different iterators:

```rust
fn main() {
    let mut r = 1..=3;
    println!("{:?}", r.next());  // ?
    println!("{:?}", r.next());  // ?
    println!("{:?}", r.next());  // ?
    println!("{:?}", r.next());  // ?
}
```

<details>
<summary>Answer</summary>

```
Some(1)
Some(2)
Some(3)
None
```

</details>

### Exercise 2: iter vs into_iter

What's the difference in output?

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];
    
    // Version 1
    for s in &v {
        println!("{}", s);
    }
    println!("{:?}", v);  // Does this work?
    
    // Version 2
    for s in v {
        println!("{}", s);
    }
    // println!("{:?}", v);  // Does this work?
}
```

<details>
<summary>Answer</summary>

```
Version 1: &v → v.iter() → s is &String. v is still usable. ✅
Version 2: v → v.into_iter() → s is String (owned). v is consumed. ❌
```

</details>

---

## Summary

| Concept | Detail |
|---------|--------|
| `Iterator` trait | Requires only `next() → Option<Item>` |
| `.iter()` | Borrows: yields `&T` |
| `.iter_mut()` | Mutably borrows: yields `&mut T` |
| `.into_iter()` | Consumes: yields `T` |
| Laziness | Adaptors don't execute until consumed |
| `for x in collection` | Calls `.into_iter()` automatically |
| `IntoIterator` | Trait for anything convertible to an iterator |

---

**Previous:** [← Closures and Ownership](./03-closures-and-ownership.md) · **Next:** [Iterator Adaptors →](./05-iterator-adaptors.md)

<p align="center"><i>Tutorial 4 of 7 — Stage 6: Closures and Iterators</i></p>

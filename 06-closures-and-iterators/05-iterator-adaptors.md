# Iterator Adaptors 🔧

> **Iterator adaptors transform one iterator into another. They're lazy (no work happens until consumed) and can be chained to build powerful data pipelines. This tutorial covers all the important adaptors: `map`, `filter`, `enumerate`, `zip`, `chain`, `take`, `skip`, `peekable`, `flat_map`, and more.**

---

## Table of Contents

- [What Are Adaptors?](#what-are-adaptors)
- [map — Transform Each Element](#map--transform-each-element)
- [filter — Keep Matching Elements](#filter--keep-matching-elements)
- [filter_map — Filter + Transform](#filter_map--filter--transform)
- [enumerate — Add Index](#enumerate--add-index)
- [zip — Pair Two Iterators](#zip--pair-two-iterators)
- [chain — Concatenate Iterators](#chain--concatenate-iterators)
- [take and skip](#take-and-skip)
- [take_while and skip_while](#take_while-and-skip_while)
- [peekable — Look Ahead](#peekable--look-ahead)
- [flat_map and flatten](#flat_map-and-flatten)
- [inspect — Debug Without Consuming](#inspect--debug-without-consuming)
- [cloned and copied](#cloned-and-copied)
- [rev — Reverse](#rev--reverse)
- [Chaining Adaptors](#chaining-adaptors)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Are Adaptors?

Adaptors take an iterator and return a **new** iterator. They don't consume anything — they're lazy:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // Each method returns a new iterator wrapping the previous
    let pipeline = v.iter()     // Iter<i32>
        .map(|x| x * 2)         // Map<Iter<i32>, ...>
        .filter(|x| x > &4)     // Filter<Map<Iter<i32>, ...>, ...>
        .take(2);                // Take<Filter<Map<Iter<i32>, ...>, ...>>
    
    // Nothing computed yet! Only when we consume:
    let result: Vec<i32> = pipeline.collect();
    println!("{:?}", result);  // [6, 8]
}
```

---

## map — Transform Each Element

Applies a closure to each element:

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    let doubled: Vec<i32> = nums.iter().map(|&x| x * 2).collect();
    println!("{:?}", doubled);  // [2, 4, 6, 8, 10]
    
    let strings: Vec<String> = nums.iter().map(|x| x.to_string()).collect();
    println!("{:?}", strings);  // ["1", "2", "3", "4", "5"]
    
    // Transform types
    let lengths: Vec<usize> = vec!["hello", "world", "hi"]
        .iter()
        .map(|s| s.len())
        .collect();
    println!("{:?}", lengths);  // [5, 5, 2]
}
```

---

## filter — Keep Matching Elements

Keeps only elements where the predicate returns `true`:

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    let evens: Vec<&i32> = nums.iter().filter(|&&x| x % 2 == 0).collect();
    println!("{:?}", evens);  // [2, 4, 6, 8, 10]
    
    // Note: filter passes &&T (double reference) because iter() gives &T
    // and filter borrows that with &
    
    // With into_iter (single reference):
    let evens: Vec<i32> = nums.into_iter().filter(|&x| x % 2 == 0).collect();
    println!("{:?}", evens);  // [2, 4, 6, 8, 10]
}
```

---

## filter_map — Filter + Transform

Combines `filter` and `map` — return `Some(value)` to keep, `None` to skip:

```rust
fn main() {
    let strings = vec!["1", "hello", "3", "world", "5"];
    
    // Parse strings, keep only valid numbers
    let numbers: Vec<i32> = strings
        .iter()
        .filter_map(|s| s.parse::<i32>().ok())  // ok() converts Result → Option
        .collect();
    
    println!("{:?}", numbers);  // [1, 3, 5]
    
    // Equivalent with filter + map (more verbose):
    let numbers: Vec<i32> = strings
        .iter()
        .map(|s| s.parse::<i32>())
        .filter(|r| r.is_ok())
        .map(|r| r.unwrap())
        .collect();
}
```

---

## enumerate — Add Index

Wraps each element with its index:

```rust
fn main() {
    let fruits = vec!["apple", "banana", "cherry"];
    
    for (i, fruit) in fruits.iter().enumerate() {
        println!("{}: {}", i, fruit);
    }
    // 0: apple
    // 1: banana
    // 2: cherry
    
    // Find index of an element
    let pos = fruits.iter().enumerate()
        .find(|(_, &f)| f == "banana")
        .map(|(i, _)| i);
    println!("banana at index: {:?}", pos);  // Some(1)
    
    // Collect into (index, value) pairs
    let indexed: Vec<(usize, &&str)> = fruits.iter().enumerate().collect();
}
```

---

## zip — Pair Two Iterators

Combines two iterators element-by-element:

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Carol"];
    let scores = vec![100, 85, 92];
    
    let pairs: Vec<(&&str, &i32)> = names.iter().zip(scores.iter()).collect();
    println!("{:?}", pairs);
    // [("Alice", 100), ("Bob", 85), ("Carol", 92)]
    
    // Stops at the shorter iterator
    let a = vec![1, 2, 3, 4, 5];
    let b = vec![10, 20, 30];
    let sums: Vec<i32> = a.iter().zip(b.iter()).map(|(x, y)| x + y).collect();
    println!("{:?}", sums);  // [11, 22, 33] (only 3 elements)
    
    // Dot product
    let a = vec![1.0, 2.0, 3.0];
    let b = vec![4.0, 5.0, 6.0];
    let dot: f64 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
    println!("Dot product: {}", dot);  // 32.0
    
    // Unzip
    let pairs = vec![(1, 'a'), (2, 'b'), (3, 'c')];
    let (nums, chars): (Vec<i32>, Vec<char>) = pairs.into_iter().unzip();
    println!("{:?} {:?}", nums, chars);  // [1, 2, 3] ['a', 'b', 'c']
}
```

---

## chain — Concatenate Iterators

Yields all elements from the first iterator, then all from the second:

```rust
fn main() {
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    
    let combined: Vec<&i32> = a.iter().chain(b.iter()).collect();
    println!("{:?}", combined);  // [1, 2, 3, 4, 5, 6]
    
    // Chain multiple
    let c = vec![7, 8, 9];
    let all: Vec<&i32> = a.iter().chain(b.iter()).chain(c.iter()).collect();
    println!("{:?}", all);  // [1, 2, 3, 4, 5, 6, 7, 8, 9]
    
    // Chain with once (add single element)
    let with_zero: Vec<i32> = std::iter::once(0)
        .chain(1..=5)
        .collect();
    println!("{:?}", with_zero);  // [0, 1, 2, 3, 4, 5]
}
```

---

## take and skip

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    // take: only first N elements
    let first_3: Vec<&i32> = v.iter().take(3).collect();
    println!("{:?}", first_3);  // [1, 2, 3]
    
    // skip: skip first N elements
    let after_3: Vec<&i32> = v.iter().skip(3).collect();
    println!("{:?}", after_3);  // [4, 5, 6, 7, 8, 9, 10]
    
    // Pagination: page 2, 3 items per page
    let page_size = 3;
    let page = 2;
    let page_data: Vec<&i32> = v.iter()
        .skip((page - 1) * page_size)
        .take(page_size)
        .collect();
    println!("Page {}: {:?}", page, page_data);  // Page 2: [4, 5, 6]
    
    // Works with infinite iterators
    let fives: Vec<i32> = std::iter::repeat(5).take(4).collect();
    println!("{:?}", fives);  // [5, 5, 5, 5]
}
```

---

## take_while and skip_while

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 4, 3, 2, 1];
    
    // take_while: take while predicate is true (stops at first false)
    let ascending: Vec<&i32> = v.iter().take_while(|&&x| x < 4).collect();
    println!("{:?}", ascending);  // [1, 2, 3]
    
    // skip_while: skip while predicate is true (starts at first false)
    let rest: Vec<&i32> = v.iter().skip_while(|&&x| x < 4).collect();
    println!("{:?}", rest);  // [4, 5, 4, 3, 2, 1]
    
    // Note: once the condition becomes false, ALL remaining elements pass through
    // It doesn't re-check — it's not the same as filter!
}
```

---

## peekable — Look Ahead

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    let mut iter = v.iter().peekable();
    
    // Peek at next element without consuming it
    println!("Peek: {:?}", iter.peek());    // Some(&&1)
    println!("Next: {:?}", iter.next());    // Some(&1)
    println!("Peek: {:?}", iter.peek());    // Some(&&2)
    println!("Peek: {:?}", iter.peek());    // Some(&&2) — still 2!
    println!("Next: {:?}", iter.next());    // Some(&2)
    
    // Useful for conditional processing
    let nums = vec![1, 2, 10, 3, 4];
    let mut iter = nums.iter().peekable();
    
    while let Some(&val) = iter.next() {
        if let Some(&&next) = iter.peek() {
            if next > val * 2 {
                println!("{} → {} (big jump!)", val, next);
            }
        }
    }
    // 2 → 10 (big jump!)
}
```

---

## flat_map and flatten

### flatten — Unwrap nested iterators

```rust
fn main() {
    let nested = vec![vec![1, 2], vec![3, 4], vec![5, 6]];
    
    let flat: Vec<&i32> = nested.iter().flatten().collect();
    println!("{:?}", flat);  // [1, 2, 3, 4, 5, 6]
    
    // Flatten Options (removes None, unwraps Some)
    let options = vec![Some(1), None, Some(3), None, Some(5)];
    let values: Vec<i32> = options.into_iter().flatten().collect();
    println!("{:?}", values);  // [1, 3, 5]
}
```

### flat_map — map + flatten

```rust
fn main() {
    let words = vec!["hello world", "foo bar baz"];
    
    let letters: Vec<&str> = words.iter()
        .flat_map(|s| s.split_whitespace())
        .collect();
    println!("{:?}", letters);
    // ["hello", "world", "foo", "bar", "baz"]
    
    // Equivalent to:
    let letters: Vec<&str> = words.iter()
        .map(|s| s.split_whitespace())
        .flatten()
        .collect();
}
```

---

## inspect — Debug Without Consuming

```rust
fn main() {
    let result: Vec<i32> = (1..=10)
        .inspect(|x| println!("before filter: {}", x))
        .filter(|x| x % 3 == 0)
        .inspect(|x| println!("after filter: {}", x))
        .map(|x| x * x)
        .inspect(|x| println!("after map: {}", x))
        .collect();
    
    println!("Result: {:?}", result);  // [9, 36, 81]
}
```

---

## cloned and copied

Convert `&T` to `T`:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // .iter() gives &i32, but we want i32
    // .copied() works for Copy types
    let owned: Vec<i32> = v.iter().copied().collect();
    
    // .cloned() works for Clone types (including Copy)
    let strings = vec![String::from("a"), String::from("b")];
    let owned: Vec<String> = strings.iter().cloned().collect();
    
    // Prefer .copied() for Copy types (no-op, more explicit)
}
```

---

## rev — Reverse

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    let reversed: Vec<&i32> = v.iter().rev().collect();
    println!("{:?}", reversed);  // [5, 4, 3, 2, 1]
    
    // Only works with DoubleEndedIterator (most collection iterators)
    for i in (0..5).rev() {
        print!("{} ", i);
    }
    // 4 3 2 1 0
}
```

---

## Chaining Adaptors

Build complex pipelines:

```rust
fn main() {
    let data = vec![
        "Alice:95", "Bob:87", "Carol:92", "Dave:78", "Eve:88",
    ];
    
    // Parse scores, filter passing (≥ 85), sort descending
    let mut honor_roll: Vec<(&str, i32)> = data.iter()
        .filter_map(|s| {
            let parts: Vec<&str> = s.split(':').collect();
            let name = parts[0];
            let score: i32 = parts[1].parse().ok()?;
            Some((name, score))
        })
        .filter(|&(_, score)| score >= 85)
        .collect();
    
    honor_roll.sort_by(|a, b| b.1.cmp(&a.1));
    
    for (name, score) in &honor_roll {
        println!("{}: {}", name, score);
    }
    // Alice: 95
    // Carol: 92
    // Eve: 88
    // Bob: 87
}
```

---

## Exercises

### Exercise 1: Pipeline

Using iterator adaptors, find the sum of squares of all odd numbers from 1 to 20:

<details>
<summary>Solution</summary>

```rust
fn main() {
    let result: i32 = (1..=20)
        .filter(|x| x % 2 != 0)
        .map(|x| x * x)
        .sum();
    
    println!("{}", result);  // 1 + 9 + 25 + 49 + 81 + 121 + 169 + 225 + 289 + 361 = 1330
}
```

</details>

### Exercise 2: Flatten and Process

Given nested vectors, flatten them, remove duplicates, and sort:

<details>
<summary>Solution</summary>

```rust
use std::collections::BTreeSet;

fn main() {
    let data = vec![vec![3, 1, 4], vec![1, 5, 9], vec![2, 6, 5]];
    
    let unique_sorted: Vec<i32> = data.into_iter()
        .flatten()
        .collect::<BTreeSet<i32>>()
        .into_iter()
        .collect();
    
    println!("{:?}", unique_sorted);
    // [1, 2, 3, 4, 5, 6, 9]
}
```

</details>

---

## Summary

| Adaptor | Purpose | Input → Output |
|---------|---------|----------------|
| `map` | Transform each element | `T → U` |
| `filter` | Keep matching elements | `T → T` (fewer) |
| `filter_map` | Filter + transform | `T → Option<U>` |
| `enumerate` | Add indices | `T → (usize, T)` |
| `zip` | Pair with another iterator | `T, U → (T, U)` |
| `chain` | Concatenate iterators | `T + T → T` |
| `take(n)` | First n elements | `T → T` (up to n) |
| `skip(n)` | Skip first n | `T → T` (after n) |
| `take_while` | Take while predicate true | `T → T` |
| `skip_while` | Skip while predicate true | `T → T` |
| `peekable` | Allow peek without consuming | `T → Peekable<T>` |
| `flatten` | Unwrap nested iterators | `Iter<Iter<T>> → Iter<T>` |
| `flat_map` | Map then flatten | `T → Iter<U>` |
| `inspect` | Debug side-effect | `T → T` (unchanged) |
| `cloned` / `copied` | `&T → T` | Reference to owned |
| `rev` | Reverse order | `T → T` |

**All adaptors are lazy — they produce new iterators without doing work.**

---

**Previous:** [← The Iterator Trait](./04-iterator-trait.md) · **Next:** [Iterator Consumers →](./06-iterator-consumers.md)

<p align="center"><i>Tutorial 5 of 7 — Stage 6: Closures and Iterators</i></p>

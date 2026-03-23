# Iterator Consumers 🍽️

> **Consumers (also called "terminal operations") are methods that actually execute the iterator pipeline and produce a final value. Without a consumer, lazy adaptors do nothing. This tutorial covers `collect`, `sum`, `count`, `fold`, `any`, `all`, `find`, `position`, `reduce`, `for_each`, and more.**

---

## Table of Contents

- [What Is a Consumer?](#what-is-a-consumer)
- [collect — Build a Collection](#collect--build-a-collection)
- [sum and product](#sum-and-product)
- [count, min, max](#count-min-max)
- [fold — The Universal Consumer](#fold--the-universal-consumer)
- [reduce](#reduce)
- [any and all](#any-and-all)
- [find and position](#find-and-position)
- [for_each](#for_each)
- [nth and last](#nth-and-last)
- [partition](#partition)
- [Comparison of Consumers](#comparison-of-consumers)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is a Consumer?

A **consumer** (or **terminal operation**) triggers the iterator to actually do work:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // Adaptors don't do anything:
    let _lazy = v.iter().map(|x| x * 2).filter(|x| x > &5);
    // ↑ Nothing happened yet!
    
    // Consumers trigger execution:
    let result: Vec<i32> = v.iter().map(|x| x * 2).filter(|x| x > &5).copied().collect();
    println!("{:?}", result);  // [6, 8, 10]
}
```

---

## collect — Build a Collection

The most versatile consumer. Builds any collection that implements `FromIterator`:

```rust
use std::collections::{HashMap, HashSet, BTreeMap, LinkedList, VecDeque};

fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    // Into Vec
    let v: Vec<i32> = nums.iter().copied().collect();
    
    // Into HashSet
    let s: HashSet<i32> = nums.iter().copied().collect();
    
    // Into String
    let s: String = "hello".chars()
        .map(|c| c.to_uppercase().next().unwrap())
        .collect();
    println!("{}", s);  // "HELLO"
    
    // Into HashMap (from tuples)
    let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2)].into_iter().collect();
    
    // Into BTreeMap
    let btree: BTreeMap<i32, i32> = (0..5).map(|x| (x, x * x)).collect();
    
    // Into VecDeque
    let deque: VecDeque<i32> = (0..5).collect();
    
    // Into LinkedList
    let list: LinkedList<i32> = (0..5).collect();
    
    // Turbofish syntax (inline type)
    let v = (0..5).collect::<Vec<i32>>();
    let v = (0..5).collect::<Vec<_>>();  // Infer element type
}
```

### Collecting Results

```rust
fn main() {
    // Collect Vec<Result<T, E>> into Result<Vec<T>, E>
    let strings = vec!["1", "2", "3"];
    let numbers: Result<Vec<i32>, _> = strings.iter().map(|s| s.parse::<i32>()).collect();
    println!("{:?}", numbers);  // Ok([1, 2, 3])
    
    // If any parse fails, whole thing is Err
    let strings = vec!["1", "oops", "3"];
    let numbers: Result<Vec<i32>, _> = strings.iter().map(|s| s.parse::<i32>()).collect();
    println!("{:?}", numbers);  // Err(ParseIntError...)
    
    // Same works with Option
    let options = vec![Some(1), Some(2), Some(3)];
    let values: Option<Vec<i32>> = options.into_iter().collect();
    println!("{:?}", values);  // Some([1, 2, 3])
    
    let options = vec![Some(1), None, Some(3)];
    let values: Option<Vec<i32>> = options.into_iter().collect();
    println!("{:?}", values);  // None
}
```

---

## sum and product

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    let total: i32 = nums.iter().sum();
    println!("Sum: {}", total);  // 15
    
    let product: i32 = nums.iter().product();
    println!("Product: {}", product);  // 120
    
    // With transformations
    let sum_of_squares: i32 = (1..=10).map(|x| x * x).sum();
    println!("Sum of squares: {}", sum_of_squares);  // 385
    
    // Floating point
    let avg: f64 = nums.iter().map(|&x| x as f64).sum::<f64>() / nums.len() as f64;
    println!("Average: {}", avg);  // 3.0
}
```

---

## count, min, max

```rust
fn main() {
    let nums = vec![3, 1, 4, 1, 5, 9, 2, 6];
    
    // count
    let n = nums.iter().count();
    println!("Count: {}", n);  // 8
    
    let even_count = nums.iter().filter(|&&x| x % 2 == 0).count();
    println!("Even count: {}", even_count);  // 3
    
    // min and max
    println!("Min: {:?}", nums.iter().min());   // Some(&1)
    println!("Max: {:?}", nums.iter().max());   // Some(&9)
    
    // Empty iterator
    let empty: Vec<i32> = vec![];
    println!("Min of empty: {:?}", empty.iter().min());  // None
    
    // min_by and max_by (custom comparison)
    let words = vec!["apple", "hi", "banana", "me"];
    let longest = words.iter().max_by_key(|s| s.len());
    println!("Longest: {:?}", longest);  // Some("banana")
    
    let shortest = words.iter().min_by_key(|s| s.len());
    println!("Shortest: {:?}", shortest);  // Some("hi")
    
    // min_by / max_by with custom comparator
    let floats = vec![1.5, 0.3, 4.2, 2.1];
    let max = floats.iter().max_by(|a, b| a.partial_cmp(b).unwrap());
    println!("Max float: {:?}", max);  // Some(4.2)
}
```

---

## fold — The Universal Consumer

`fold` accumulates a value, starting from an initial value:

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    // sum via fold
    let sum = nums.iter().fold(0, |acc, &x| acc + x);
    println!("Sum: {}", sum);  // 15
    
    // How it works step by step:
    // fold(0, |acc, &x| acc + x)
    // acc=0, x=1 → 0 + 1 = 1
    // acc=1, x=2 → 1 + 2 = 3
    // acc=3, x=3 → 3 + 3 = 6
    // acc=6, x=4 → 6 + 4 = 10
    // acc=10, x=5 → 10 + 5 = 15
    
    // product via fold
    let product = nums.iter().fold(1, |acc, &x| acc * x);
    println!("Product: {}", product);  // 120
    
    // Build a string
    let sentence = ["hello", "beautiful", "world"].iter()
        .fold(String::new(), |mut acc, &word| {
            if !acc.is_empty() {
                acc.push(' ');
            }
            acc.push_str(word);
            acc
        });
    println!("{}", sentence);  // "hello beautiful world"
    
    // Count occurrences
    let text = "hello world";
    let vowel_count = text.chars()
        .fold(0, |count, c| {
            if "aeiou".contains(c) { count + 1 } else { count }
        });
    println!("Vowels: {}", vowel_count);  // 3
    
    // Fold can build any type
    let max = nums.iter().fold(i32::MIN, |acc, &x| if x > acc { x } else { acc });
    println!("Max: {}", max);  // 5
}
```

---

## reduce

Like `fold` but uses the first element as the initial value:

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    // reduce: no initial value needed
    let sum = nums.iter().copied().reduce(|acc, x| acc + x);
    println!("{:?}", sum);  // Some(15)
    
    // Returns None for empty iterators
    let empty: Vec<i32> = vec![];
    let sum = empty.iter().copied().reduce(|acc, x| acc + x);
    println!("{:?}", sum);  // None
    
    // Find longest string
    let words = vec!["apple", "banana", "cherry"];
    let longest = words.into_iter().reduce(|a, b| if a.len() >= b.len() { a } else { b });
    println!("{:?}", longest);  // Some("cherry" or "banana")
}
```

---

## any and all

Short-circuit boolean checks:

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    // any: returns true if ANY element matches
    let has_even = nums.iter().any(|&x| x % 2 == 0);
    println!("Has even: {}", has_even);  // true
    
    let has_negative = nums.iter().any(|&x| x < 0);
    println!("Has negative: {}", has_negative);  // false
    
    // all: returns true if ALL elements match
    let all_positive = nums.iter().all(|&x| x > 0);
    println!("All positive: {}", all_positive);  // true
    
    let all_even = nums.iter().all(|&x| x % 2 == 0);
    println!("All even: {}", all_even);  // false
    
    // Short-circuit: stops as soon as answer is known
    let result = (0..).any(|x| {
        println!("Checking {}", x);
        x == 3
    });
    // Prints: Checking 0, 1, 2, 3 — stops at 3
    
    // Empty iterator
    let empty: Vec<i32> = vec![];
    println!("any on empty: {}", empty.iter().any(|&x| x > 0));  // false
    println!("all on empty: {}", empty.iter().all(|&x| x > 0));  // true (vacuously)
}
```

---

## find and position

```rust
fn main() {
    let nums = vec![1, 3, 5, 8, 10, 13];
    
    // find: first element matching predicate
    let first_even = nums.iter().find(|&&x| x % 2 == 0);
    println!("{:?}", first_even);  // Some(&8)
    
    // find_map: find + transform
    let first_big_even = nums.iter()
        .find_map(|&x| if x % 2 == 0 && x > 5 { Some(x * 10) } else { None });
    println!("{:?}", first_big_even);  // Some(80)
    
    // position: index of first match
    let pos = nums.iter().position(|&x| x > 5);
    println!("First > 5 at index: {:?}", pos);  // Some(3)
    
    // rposition: index of LAST match (searches from end)
    let rpos = nums.iter().rposition(|&x| x % 2 != 0);
    println!("Last odd at index: {:?}", rpos);  // Some(5)
}
```

---

## for_each

Like `for`, but as a method — useful at the end of chains:

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5];
    
    // for_each
    nums.iter()
        .filter(|&&x| x % 2 != 0)
        .for_each(|x| println!("Odd: {}", x));
    // Odd: 1
    // Odd: 3
    // Odd: 5
    
    // Equivalent to:
    for x in nums.iter().filter(|&&x| x % 2 != 0) {
        println!("Odd: {}", x);
    }
}
```

---

## nth and last

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];
    
    // nth: get element at index (consumes elements up to that point)
    let mut iter = v.iter();
    println!("{:?}", iter.nth(2));   // Some(&30) — consumed 10, 20, 30
    println!("{:?}", iter.nth(0));   // Some(&40) — next element (index 0 from current position)
    
    // last: get final element (consumes entire iterator)
    let last = v.iter().last();
    println!("{:?}", last);  // Some(&50)
    
    // last of empty
    let empty: Vec<i32> = vec![];
    println!("{:?}", empty.iter().last());  // None
}
```

---

## partition

Split into two collections based on a predicate:

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    let (evens, odds): (Vec<i32>, Vec<i32>) = nums
        .into_iter()
        .partition(|&x| x % 2 == 0);
    
    println!("Evens: {:?}", evens);  // [2, 4, 6, 8, 10]
    println!("Odds: {:?}", odds);    // [1, 3, 5, 7, 9]
}
```

---

## Comparison of Consumers

| Consumer | Returns | Short-circuits | Purpose |
|----------|---------|---------------|---------|
| `collect()` | Collection | No | Build a collection |
| `sum()` | Number | No | Add all elements |
| `product()` | Number | No | Multiply all elements |
| `count()` | `usize` | No | Count elements |
| `min()` / `max()` | `Option<T>` | No | Extreme values |
| `fold(init, f)` | Any type | No | General accumulation |
| `reduce(f)` | `Option<T>` | No | Fold without initial value |
| `any(f)` | `bool` | Yes | Any match? |
| `all(f)` | `bool` | Yes | All match? |
| `find(f)` | `Option<&T>` | Yes | First match |
| `position(f)` | `Option<usize>` | Yes | Index of first match |
| `for_each(f)` | `()` | No | Side effects |
| `nth(n)` | `Option<T>` | Yes | Element at index |
| `last()` | `Option<T>` | No | Final element |
| `partition(f)` | `(C, C)` | No | Split into two |

---

## Exercises

### Exercise 1: Word Frequency

Count word frequencies and find the most common word:

```rust
fn most_common_word(text: &str) -> Option<(String, usize)> {
    todo!()
}
```

<details>
<summary>Solution</summary>

```rust
use std::collections::HashMap;

fn most_common_word(text: &str) -> Option<(String, usize)> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    
    counts.into_iter()
        .max_by_key(|&(_, count)| count)
        .map(|(word, count)| (word, count))
}

fn main() {
    let text = "the cat sat on the mat the cat";
    println!("{:?}", most_common_word(text));
    // Some(("the", 3))
}
```

</details>

### Exercise 2: Running Average

Compute running averages using fold:

```rust
fn running_averages(nums: &[f64]) -> Vec<f64> {
    todo!()
}
// running_averages(&[1.0, 2.0, 3.0, 4.0]) → [1.0, 1.5, 2.0, 2.5]
```

<details>
<summary>Solution</summary>

```rust
fn running_averages(nums: &[f64]) -> Vec<f64> {
    let (averages, _, _) = nums.iter().fold(
        (Vec::new(), 0.0, 0_usize),
        |(mut avgs, sum, count), &x| {
            let new_sum = sum + x;
            let new_count = count + 1;
            avgs.push(new_sum / new_count as f64);
            (avgs, new_sum, new_count)
        },
    );
    averages
}

fn main() {
    let result = running_averages(&[1.0, 2.0, 3.0, 4.0]);
    println!("{:?}", result);  // [1.0, 1.5, 2.0, 2.5]
}
```

</details>

---

## Summary

| Consumer | Use When |
|----------|----------|
| `collect()` | Building any collection from an iterator |
| `sum()` / `product()` | Numeric aggregation |
| `fold()` | Custom accumulation with initial value |
| `reduce()` | Custom accumulation, first element as initial |
| `any()` / `all()` | Boolean checks (short-circuits) |
| `find()` | First matching element |
| `count()` | Number of elements |
| `min()` / `max()` | Extreme values |
| `for_each()` | Execute side effects |
| `partition()` | Split into two groups |

**Remember: adaptors are lazy, consumers trigger execution.**

---

**Previous:** [← Iterator Adaptors](./05-iterator-adaptors.md) · **Next:** [Custom Iterators →](./07-custom-iterators.md)

<p align="center"><i>Tutorial 6 of 7 — Stage 6: Closures and Iterators</i></p>

# Sets — HashSet and BTreeSet 🎯

> **Sets store unique values with no duplicates. `HashSet<T>` provides O(1) operations; `BTreeSet<T>` keeps elements sorted. Both support mathematical set operations: union, intersection, difference, and symmetric difference.**

---

## Table of Contents

- [HashSet Basics](#hashset-basics)
- [BTreeSet Basics](#btreeset-basics)
- [Adding and Removing](#adding-and-removing)
- [Checking Membership](#checking-membership)
- [Set Operations](#set-operations)
- [Subset and Superset](#subset-and-superset)
- [Iterating](#iterating)
- [Common Patterns](#common-patterns)
- [HashSet vs BTreeSet](#hashset-vs-btreeset)
- [Exercises](#exercises)
- [Summary](#summary)

---

## HashSet Basics

```rust
use std::collections::HashSet;

fn main() {
    // Empty
    let mut fruits: HashSet<&str> = HashSet::new();
    
    // From array
    let fruits: HashSet<&str> = HashSet::from(["apple", "banana", "cherry"]);
    
    // From iterator
    let nums: HashSet<i32> = (0..10).collect();
    
    // From a vec (removes duplicates!)
    let data = vec![1, 2, 3, 2, 1, 4, 3, 5];
    let unique: HashSet<i32> = data.into_iter().collect();
    println!("{:?}", unique);  // {1, 2, 3, 4, 5} (unordered)
    
    // With capacity
    let set: HashSet<String> = HashSet::with_capacity(100);
}
```

---

## BTreeSet Basics

Same API, but sorted:

```rust
use std::collections::BTreeSet;

fn main() {
    let data = vec![5, 3, 1, 4, 2, 3, 1];
    let sorted_unique: BTreeSet<i32> = data.into_iter().collect();
    
    // Always iterated in sorted order
    for n in &sorted_unique {
        print!("{} ", n);
    }
    println!();
    // 1 2 3 4 5
    
    // First / last
    println!("Min: {:?}", sorted_unique.first());  // Some(1)
    println!("Max: {:?}", sorted_unique.last());    // Some(5)
    
    // Range queries
    for n in sorted_unique.range(2..=4) {
        print!("{} ", n);
    }
    println!();
    // 2 3 4
}
```

---

## Adding and Removing

```rust
use std::collections::HashSet;

fn main() {
    let mut set = HashSet::new();
    
    // insert() returns true if the value was new
    println!("{}", set.insert("apple"));   // true  (added)
    println!("{}", set.insert("banana"));  // true  (added)
    println!("{}", set.insert("apple"));   // false (already existed)
    
    // remove() returns true if the value was present
    println!("{}", set.remove("banana"));  // true  (removed)
    println!("{}", set.remove("cherry"));  // false (wasn't there)
    
    // take() removes and returns the value
    let taken = set.take("apple");
    println!("{:?}", taken);  // Some("apple")
    
    // replace() inserts and returns the old value if it existed
    set.insert("apple");
    let old = set.replace("apple");
    println!("{:?}", old);  // Some("apple")
    
    // retain() keeps only elements matching predicate
    let mut nums: HashSet<i32> = (0..10).collect();
    nums.retain(|&x| x % 2 == 0);
    println!("{:?}", nums);  // {0, 2, 4, 6, 8}
    
    // clear
    nums.clear();
    println!("{}", nums.is_empty());  // true
}
```

---

## Checking Membership

```rust
use std::collections::HashSet;

fn main() {
    let fruits: HashSet<&str> = HashSet::from(["apple", "banana", "cherry"]);
    
    // contains
    println!("{}", fruits.contains("apple"));   // true
    println!("{}", fruits.contains("grape"));   // false
    
    // len and is_empty
    println!("{}", fruits.len());               // 3
    println!("{}", fruits.is_empty());          // false
}
```

---

## Set Operations

```
A = {1, 2, 3, 4}
B = {3, 4, 5, 6}

Union:                A ∪ B = {1, 2, 3, 4, 5, 6}
Intersection:         A ∩ B = {3, 4}
Difference:           A \ B = {1, 2}           (in A but not B)
Symmetric Difference: A △ B = {1, 2, 5, 6}    (in one but not both)
```

```rust
use std::collections::HashSet;

fn main() {
    let a: HashSet<i32> = [1, 2, 3, 4].into();
    let b: HashSet<i32> = [3, 4, 5, 6].into();
    
    // Union: elements in either set
    let union: HashSet<&i32> = a.union(&b).collect();
    println!("Union: {:?}", union);
    // {1, 2, 3, 4, 5, 6}
    
    // Intersection: elements in both sets
    let inter: HashSet<&i32> = a.intersection(&b).collect();
    println!("Intersection: {:?}", inter);
    // {3, 4}
    
    // Difference: in A but not in B
    let diff: HashSet<&i32> = a.difference(&b).collect();
    println!("A - B: {:?}", diff);
    // {1, 2}
    
    // Symmetric difference: in one but not both
    let sym_diff: HashSet<&i32> = a.symmetric_difference(&b).collect();
    println!("A △ B: {:?}", sym_diff);
    // {1, 2, 5, 6}
    
    // Using operators (requires & references)
    let union: HashSet<i32> = &a | &b;         // Union
    let inter: HashSet<i32> = &a & &b;         // Intersection
    let diff: HashSet<i32> = &a - &b;          // Difference
    let sym_diff: HashSet<i32> = &a ^ &b;      // Symmetric difference
}
```

---

## Subset and Superset

```rust
use std::collections::HashSet;

fn main() {
    let a: HashSet<i32> = [1, 2, 3].into();
    let b: HashSet<i32> = [1, 2, 3, 4, 5].into();
    
    // Is A a subset of B? (every element of A is in B)
    println!("A ⊆ B: {}", a.is_subset(&b));      // true
    
    // Is B a superset of A?
    println!("B ⊇ A: {}", b.is_superset(&a));    // true
    
    // Are they disjoint? (no elements in common)
    let c: HashSet<i32> = [10, 20].into();
    println!("A ∩ C = ∅: {}", a.is_disjoint(&c)); // true
    println!("A ∩ B = ∅: {}", a.is_disjoint(&b)); // false
}
```

---

## Iterating

```rust
use std::collections::HashSet;

fn main() {
    let set: HashSet<&str> = ["rust", "go", "python"].into();
    
    // Iterate (no guaranteed order for HashSet)
    for item in &set {
        println!("{}", item);
    }
    
    // into_iter (consumes)
    for item in set {
        println!("{}", item);
    }
    
    // Convert to sorted Vec
    let set: HashSet<i32> = [3, 1, 4, 1, 5].into();
    let mut sorted: Vec<&i32> = set.iter().collect();
    sorted.sort();
    println!("{:?}", sorted);  // [1, 3, 4, 5]
    
    // Or just use BTreeSet for sorted iteration
    use std::collections::BTreeSet;
    let sorted: BTreeSet<i32> = [3, 1, 4, 1, 5].into();
    for n in &sorted {
        print!("{} ", n);
    }
    // 1 3 4 5
}
```

---

## Common Patterns

```rust
use std::collections::HashSet;

fn main() {
    // Remove duplicates from Vec, preserving order
    let data = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5];
    let mut seen = HashSet::new();
    let unique: Vec<i32> = data
        .into_iter()
        .filter(|x| seen.insert(*x))  // insert returns true if new
        .collect();
    println!("{:?}", unique);  // [3, 1, 4, 5, 9, 2, 6]
    
    // Find duplicates
    let data = vec![1, 2, 3, 2, 4, 3, 5];
    let mut seen = HashSet::new();
    let duplicates: HashSet<i32> = data
        .into_iter()
        .filter(|x| !seen.insert(*x))
        .collect();
    println!("Duplicates: {:?}", duplicates);  // {2, 3}
    
    // Tags / categories
    let mut tags: HashSet<String> = HashSet::new();
    tags.insert("rust".into());
    tags.insert("programming".into());
    tags.insert("tutorial".into());
    
    let required: HashSet<String> = ["rust", "tutorial"]
        .into_iter()
        .map(String::from)
        .collect();
    
    if required.is_subset(&tags) {
        println!("All required tags present!");
    }
}
```

---

## HashSet vs BTreeSet

| Feature | `HashSet` | `BTreeSet` |
|---------|-----------|------------|
| Insert/lookup | O(1) avg | O(log n) |
| Sorted iteration | No | Yes |
| Range queries | No | Yes |
| Min/Max | No (O(n)) | Yes (O(log n)) |
| Element requirement | `Hash + Eq` | `Ord` |
| Use when | Fast membership test | Need ordering |

---

## Exercises

### Exercise 1: Common Elements

Write a function that finds common elements across multiple vectors:

```rust
fn common_elements(lists: &[Vec<i32>]) -> Vec<i32> {
    todo!()
}
// common_elements(&[vec![1,2,3], vec![2,3,4], vec![3,4,5]]) → [3]
```

<details>
<summary>Solution</summary>

```rust
use std::collections::HashSet;

fn common_elements(lists: &[Vec<i32>]) -> Vec<i32> {
    if lists.is_empty() {
        return vec![];
    }
    
    let mut result: HashSet<i32> = lists[0].iter().copied().collect();
    
    for list in &lists[1..] {
        let set: HashSet<i32> = list.iter().copied().collect();
        result = &result & &set;
    }
    
    let mut v: Vec<i32> = result.into_iter().collect();
    v.sort();
    v
}

fn main() {
    let result = common_elements(&[
        vec![1, 2, 3, 4],
        vec![2, 3, 4, 5],
        vec![3, 4, 5, 6],
    ]);
    println!("{:?}", result);  // [3, 4]
}
```

</details>

---

## Summary

| Type | Key Trait | Lookup | Ordered | Best For |
|------|-----------|--------|---------|----------|
| `HashSet<T>` | `Hash + Eq` | O(1) | No | Fast membership, dedup |
| `BTreeSet<T>` | `Ord` | O(log n) | Yes | Sorted, ranges, min/max |

Set operations: `union`, `intersection`, `difference`, `symmetric_difference`

Operators: `|` (union), `&` (intersection), `-` (difference), `^` (symmetric difference)

---

**Previous:** [← BTreeMap](./05-btreemap.md) · **Next:** [VecDeque and Others →](./07-vecdeque-and-others.md)

<p align="center"><i>Tutorial 6 of 8 — Stage 5: Collections</i></p>

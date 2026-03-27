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

### Set Theory — The Mathematical Foundation

Before diving deeper into Rust's API, it's worth understanding **where sets come from** — because the operations you'll use in code map directly to mathematical notation invented over 150 years ago.

**Georg Cantor (1874)** defined a set as *"a collection of definite, distinct objects of our perception or thought."* This deceptively simple idea became the **foundation of all modern mathematics**. Every branch — algebra, topology, analysis, logic — is built on set theory. When you use a `HashSet` in Rust, you're wielding one of math's most powerful abstractions.

The fundamental property: **NO DUPLICATES.** Every element in a set is unique. Insert the same value twice, and the set doesn't change. This single constraint enables everything below.

**The Four Core Operations (with Venn Diagrams):**

```
  Union (A ∪ B)           Intersection (A ∩ B)      Difference (A \ B)
 ┌─────────────────┐     ┌─────────────────┐      ┌─────────────────┐
 │  ┌──A───┐       │     │  ┌──A───┐       │      │  ┌──A───┐       │
 │  │██████│██B──┐ │     │  │      │██B──┐ │      │  │██████│  B──┐ │
 │  │██████│█████│ │     │  │      │█████│ │      │  │██████│     │ │
 │  │██████│█████│ │     │  │      │█████│ │      │  │██████│     │ │
 │  └──────│█████│ │     │  └──────│     │ │      │  └──────│     │ │
 │         └─────┘ │     │         └─────┘ │      │         └─────┘ │
 └─────────────────┘     └─────────────────┘      └─────────────────┘
  Everything in           Only the overlap          In A, but NOT in B
  either A or B

  Symmetric Difference (A △ B)
 ┌─────────────────┐
 │  ┌──A───┐       │
 │  │██████│  B──┐ │
 │  │██████│     │ │
 │  │██████│█████│ │
 │  └──────│█████│ │
 │         └─────┘ │
 └─────────────────┘
  In one, but NOT both
```

**How This Maps to Programming:**

| Math Concept | Programming Use Case | Example |
|---|---|---|
| A ∪ B (union) | Combine tag lists, merge results | All skills from two job postings |
| A ∩ B (intersection) | Find commonalities | Mutual friends between two users |
| A \ B (difference) | Find what's missing | Files in repo but not in backup |
| A △ B (symmetric diff) | Find mismatches | Config drift between two servers |
| x ∈ A (membership) | Existence checks | Is this user ID in the banned set? |
| \|A\| (cardinality) | Count unique items | Unique visitors to a website |

The no-duplicates guarantee makes sets the **go-to data structure** for:
- **Deduplication** — pour data in, duplicates vanish
- **O(1) membership testing** — "have I seen this before?" in constant time
- **Finding commonalities** — intersection of two datasets in linear time

Every set operation you'll see in Rust — `.union()`, `.intersection()`, `.difference()`, `.symmetric_difference()` — is a direct implementation of Cantor's original ideas. The math is the API.

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

### Sets Across Programming Languages

Rust's `HashSet` and `BTreeSet` don't exist in a vacuum. Every major language has a set abstraction, but the design trade-offs differ in revealing ways:

| Language | Hash Set | Sorted Set | Literal Syntax | Operator Support | Lazy Operations? |
|---|---|---|---|---|---|
| **Rust** | `HashSet<T>` | `BTreeSet<T>` | No | `\|`, `&`, `-`, `^` (on refs) | Yes (iterators) |
| **Python** | `set` / `frozenset` | — (use `sorted()`) | `{1, 2, 3}` | `\|`, `&`, `-`, `^` | No (returns new set) |
| **Java** | `HashSet<T>` | `TreeSet<T>` | No | No | No |
| **C++** | `unordered_set<T>` | `set<T>` (Red-Black tree) | No (C++11 init list) | No | No |
| **JavaScript** | `Set` (ES6) | — | No | `union()`, `intersection()` (2024) | No |
| **Go** | — (use `map[T]struct{}`) | — | No | No | No |

**Notable differences:**

**Python** has the best set ergonomics of any language. Literal syntax (`{1, 2, 3}`), full operator support (`a | b`, `a & b`, `a - b`, `a ^ b`), and `frozenset` for immutable/hashable sets. The gold standard for readability:

```python
mutual = alice_friends & bob_friends   # intersection
all_tags = post1_tags | post2_tags      # union
```

**Go** has no built-in set type at all. The idiomatic workaround is `map[T]struct{}` — using an empty struct as the value to consume zero extra bytes. This is conceptually identical to Rust's trick where `HashSet<T>` is backed by `HashMap<T, ()>`.

```go
seen := map[string]struct{}{}
seen["apple"] = struct{}{}     // insert
_, ok := seen["apple"]         // membership test
```

**JavaScript's** `Set` (ES6, 2015) had no set algebra methods for nearly a decade. You had to write manual loops for union/intersection. The 2024 TC39 proposal finally added `.union()`, `.intersection()`, `.difference()`, and `.symmetricDifference()` — nine years after `Set` was introduced.

**Rust's unique advantage — laziness.** When you call `.union(&b)` in Rust, it returns an **iterator**, not a new set. Nothing is computed until you consume it. This means you can chain operations without allocating intermediate collections:

```rust
// Lazy: no allocation until .collect()
let result: HashSet<_> = a.union(&b)
    .filter(|&&x| x > 0)
    .copied()
    .collect();
```

Python's equivalent (`a | b`) allocates a brand new set immediately. For large sets or chained operations, Rust's lazy approach can save significant memory and time.

**The operator syntax** in Rust requires references (`&a | &b`), which returns an owned `HashSet`. The method syntax (`.union(&b)`) returns an iterator of references. Both are valid — operators for convenience, methods for lazy composition.

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

### When Sets Shine — Real-World Use Cases

Knowing the API is one thing. Knowing **when to reach for a set** is the skill that separates experienced Rustaceans from beginners.

**1. Instant Deduplication**

```rust
let data = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3];
let unique: HashSet<i32> = data.into_iter().collect();
// Done. All duplicates gone.
```

⚠️ **But this loses insertion order!** If you need uniqueness + original order, the `indexmap` crate provides `IndexSet`:

```rust
use indexmap::IndexSet;
let ordered_unique: IndexSet<i32> = vec![3, 1, 4, 1, 5].into_iter().collect();
// [3, 1, 4, 5] — unique AND in original order
```

**2. The "Seen Set" Pattern**

The most common set pattern in real code — tracking what you've already processed:

```rust
let mut seen = HashSet::new();
for item in stream_of_events {
    if seen.insert(item.id) {   // returns true if NEW
        process(item);           // only process each ID once
    }
}
```

This appears everywhere: crawlers (visited URLs), event processing (dedup by ID), graph traversal (visited nodes in BFS/DFS).

**3. Membership Testing in Hot Loops**

```
Performance comparison — "Is x in the collection?"
┌────────────────┬────────────┬───────────────┐
│ Collection     │ 10 items   │ 1,000 items   │
├────────────────┼────────────┼───────────────┤
│ Vec::contains  │ ~10 ns     │ ~1,000 ns     │
│ HashSet::contains │ ~10 ns │ ~10 ns        │
└────────────────┴────────────┴───────────────┘
```

`Vec::contains()` is O(n) — it scans every element. `HashSet::contains()` is O(1). For 1,000 elements, that's roughly **100× faster**. If you're checking membership inside a loop, convert to a `HashSet` first.

**4. Permission & Access Control**

```rust
let user_perms: HashSet<&str> = ["read", "write", "delete"].into();
let required: HashSet<&str> = ["read", "write"].into();

if required.is_subset(&user_perms) {
    println!("Access granted");
}
```

This pattern is used in authorization systems, feature flags, and capability-based security.

**5. Compiler / Linter Internals**

Compilers use sets extensively: tracking defined variables, checking for unused imports, detecting duplicate definitions, computing live variable sets for optimization passes.

**When NOT to Use a Set:**

| Need | Don't Use | Use Instead |
|---|---|---|
| Uniqueness + sorted order | `HashSet` | `BTreeSet` |
| Uniqueness + insertion order | `HashSet` | `IndexSet` (indexmap crate) |
| Tiny collection (< 10 items) | `HashSet` | `Vec` + `.contains()` |
| Counting occurrences | `HashSet` | `HashMap<T, usize>` |

**Memory overhead:** `HashSet` carries ~88 bytes of base overhead plus per-entry costs (key + hash + metadata). For very small collections (under ~10 elements), a plain `Vec` with `.contains()` can actually be **faster** due to CPU cache locality — the entire `Vec` fits in a cache line, while `HashSet` entries are scattered across memory. Profile before optimizing, but be aware of the crossover point.

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

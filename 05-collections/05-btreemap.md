# BTreeMap — Sorted Maps 🌳

> **`BTreeMap<K, V>` is an ordered map backed by a B-tree. Keys are always sorted, you can do range queries, and all operations are O(log n). Use it when you need sorted iteration or range-based lookups.**

---

## Table of Contents

- [BTreeMap vs HashMap](#btreemap-vs-hashmap)
- [Creating a BTreeMap](#creating-a-btreemap)
- [Basic Operations](#basic-operations)
- [Sorted Iteration](#sorted-iteration)
- [Range Queries](#range-queries)
- [Entry API](#entry-api)
- [When to Use BTreeMap](#when-to-use-btreemap)
- [Exercises](#exercises)
- [Summary](#summary)

---

## BTreeMap vs HashMap

| Feature | `HashMap` | `BTreeMap` |
|---------|-----------|------------|
| Ordering | Unordered | Sorted by key |
| Lookup | O(1) average | O(log n) |
| Insert | O(1) average | O(log n) |
| Range queries | No | Yes |
| Key requirement | `Hash + Eq` | `Ord` |
| Worst case | O(n) | O(log n) |
| Memory | Hash table (sparse) | B-tree nodes (dense) |
| Cache friendly | Less | More (data locality) |

---

## B-Trees — The Data Structure That Powers Databases

B-Trees were invented in **1970** by **Rudolf Bayer** and **Ed McCreight** at **Boeing Research Labs**. The "B" in B-Tree? Nobody knows for sure — it might stand for Boeing, Bayer, balanced, or bushy. Bayer himself never clarified it.

But here's the remarkable thing: **virtually every major database in the world uses B-Trees**. PostgreSQL, MySQL, SQLite, MongoDB — they all rely on B-Tree indexes. It's arguably the most important data structure in systems programming.

### Why Databases Love B-Trees

The killer feature is **fanout** — each node stores *many* keys, not just one. This minimizes disk I/O:

```text
            Binary Search Tree              B-Tree (order 4)
            ─────────────────              ─────────────────
                  [50]                      [20 | 50 | 80]
                 /    \                   /    |     |    \
              [25]    [75]          [5|10] [30|40] [60|70] [90|95]
             /  \     /  \
          [12] [37] [62] [87]       Each node = one disk page (4KB)
           ...  ...  ...  ...       One disk seek reads MANY keys

  Height ≈ log₂(n)                  Height ≈ log_B(n)
  1 key per node → many seeks       B keys per node → few seeks
```

A typical disk page is **4KB**. A B-Tree node is designed to fit in exactly one page. Reading one node = one disk seek. So the fewer nodes you traverse, the fewer disk reads — and B-Trees minimize that beautifully.

### The Numbers Tell the Story

For **1 million keys**:

| Structure | Keys/Node | Height (1M keys) | Cache Friendly? | Primary Use |
|-----------|-----------|-------------------|-----------------|-------------|
| Binary Search Tree | 1 | ~20 levels | No (pointer chasing) | Textbook examples |
| Red-Black Tree | 1 | ~20 levels | No (pointer chasing) | `TreeMap` in Java/C++ |
| B-Tree (order 100) | up to 199 | ~3 levels | Yes (sequential keys) | Disk-based databases |
| B-Tree (order 6) | up to 11 | ~5 levels | Yes (sequential keys) | Rust's `BTreeMap` |

Rust's `BTreeMap` uses **B=6**, meaning each internal node holds up to **11 keys**. This is smaller than a database B-Tree (which might use B=100+) because `BTreeMap` is optimized for **in-memory** use rather than disk. With B=6, each node fits comfortably in a CPU cache line, giving you excellent data locality without the overhead of managing huge nodes.

### From Disk Pages to Cache Lines

```text
  Database B-Tree (B ≈ 100+)        Rust BTreeMap (B = 6)
  ──────────────────────────        ─────────────────────
  Node = 4KB disk page              Node = fits in cache line(s)
  Goal: minimize disk seeks          Goal: minimize cache misses
  Used by: PostgreSQL, MySQL         Used by: your Rust programs
  Same idea, different scale!
```

The genius of the B-Tree is that the same fundamental idea — "store many keys per node to reduce lookups" — works at every level of the memory hierarchy: disk, RAM, and CPU cache.

---

## Creating a BTreeMap

```rust
use std::collections::BTreeMap;

fn main() {
    // Empty
    let mut map: BTreeMap<String, i32> = BTreeMap::new();
    
    // From array of tuples
    let scores = BTreeMap::from([
        ("Alice", 100),
        ("Bob", 85),
        ("Carol", 92),
    ]);
    
    // From iterator
    let map: BTreeMap<i32, i32> = (0..5).map(|x| (x, x * x)).collect();
    println!("{:?}", map);
    // {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
}
```

---

## Basic Operations

All the same methods as HashMap:

```rust
use std::collections::BTreeMap;

fn main() {
    let mut map = BTreeMap::new();
    
    // Insert
    map.insert("cherry", 3);
    map.insert("apple", 1);
    map.insert("banana", 2);
    
    // Get
    println!("{:?}", map.get("apple"));    // Some(1)
    println!("{}", map["banana"]);          // 2
    
    // Contains
    println!("{}", map.contains_key("cherry"));  // true
    
    // Remove
    let removed = map.remove("banana");
    println!("{:?}", removed);  // Some(2)
    
    // Length
    println!("{}", map.len());  // 2
}
```

---

## Sorted Iteration

**This is the key advantage.** BTreeMap always iterates in sorted key order:

```rust
use std::collections::BTreeMap;

fn main() {
    let mut map = BTreeMap::new();
    map.insert(5, "five");
    map.insert(1, "one");
    map.insert(3, "three");
    map.insert(2, "two");
    map.insert(4, "four");
    
    // Always sorted by key!
    for (k, v) in &map {
        print!("{}: {}  ", k, v);
    }
    println!();
    // 1: one  2: two  3: three  4: four  5: five
    
    // First and last
    println!("First: {:?}", map.first_key_value());  // Some((1, "one"))
    println!("Last: {:?}", map.last_key_value());    // Some((5, "five"))
    
    // Pop first/last (remove and return)
    let first = map.pop_first();
    println!("Popped: {:?}", first);  // Some((1, "one"))
    
    let last = map.pop_last();
    println!("Popped: {:?}", last);   // Some((5, "five"))
}
```

---

## Range Queries

```rust
use std::collections::BTreeMap;
use std::ops::Bound;

fn main() {
    let mut scores: BTreeMap<i32, &str> = BTreeMap::new();
    scores.insert(60, "Carol");
    scores.insert(72, "Dave");
    scores.insert(85, "Bob");
    scores.insert(92, "Alice");
    scores.insert(100, "Eve");
    
    // Range: 70..=90
    println!("Scores 70-90:");
    for (score, name) in scores.range(70..=90) {
        println!("  {}: {}", name, score);
    }
    // Bob: 85
    // (only entries with keys in 70..=90)
    
    // Range with different bound types
    use std::ops::Bound::*;
    for (score, name) in scores.range((Excluded(60), Included(92))) {
        println!("  {}: {}", name, score);
    }
    // 72: Dave, 85: Bob, 92: Alice
    
    // From a key to the end
    println!("Scores >= 85:");
    for (score, name) in scores.range(85..) {
        println!("  {}: {}", name, score);
    }
    
    // Mutable range
    let mut map: BTreeMap<i32, i32> = (0..10).map(|x| (x, x)).collect();
    for (_, v) in map.range_mut(3..7) {
        *v *= 10;
    }
    println!("{:?}", map);
    // {0: 0, 1: 1, 2: 2, 3: 30, 4: 40, 5: 50, 6: 60, 7: 7, 8: 8, 9: 9}
}
```

---

## Entry API

Same entry API as HashMap:

```rust
use std::collections::BTreeMap;

fn main() {
    let text = "hello world hello rust";
    let mut word_count: BTreeMap<&str, i32> = BTreeMap::new();
    
    for word in text.split_whitespace() {
        word_count.entry(word)
            .and_modify(|c| *c += 1)
            .or_insert(1);
    }
    
    // Printed in alphabetical order!
    for (word, count) in &word_count {
        println!("{}: {}", word, count);
    }
    // hello: 2
    // rust: 1
    // world: 1
}
```

---

## When to Use BTreeMap

| Use Case | Best Choice |
|----------|-------------|
| Fast lookup, no ordering needed | `HashMap` |
| Need sorted iteration | `BTreeMap` |
| Range queries | `BTreeMap` |
| Need min/max key | `BTreeMap` |
| Key type doesn't implement `Hash` | `BTreeMap` |
| Predictable performance (no worst-case O(n)) | `BTreeMap` |
| Small maps (< ~20 entries) | `BTreeMap` (often faster due to cache) |
| Large maps with random access | `HashMap` |

### Sorted Collections in Other Languages

Rust's `BTreeMap` is an unusual choice — most languages use **Red-Black trees** for their sorted maps. Here's the landscape:

| Language | Sorted Map Type | Underlying Structure | Cache Friendly? |
|----------|----------------|----------------------|----------------|
| **Rust** | `BTreeMap<K,V>` | B-Tree (B=6) | Yes — 11 keys per node, contiguous |
| **Java** | `TreeMap<K,V>` | Red-Black tree | No — 1 key per node, pointer chasing |
| **C++** | `std::map<K,V>` | Red-Black tree | No — 1 key per node, pointer chasing |
| **C#** | `SortedDictionary<K,V>` | Red-Black tree | No — 1 key per node |
| **Python** | ❌ None built-in | Use `sortedcontainers.SortedDict` | Varies |
| **JavaScript** | ❌ None built-in | `Map` is insertion-ordered, not sorted | N/A |
| **Go** | ❌ None built-in | `map` is unordered hash table | N/A |

**Python** is a notable gap — `dict` preserves insertion order (since 3.7), but there's no built-in sorted map. You need the third-party `sortedcontainers` library. **JavaScript** and **Go** have the same problem — no sorted map out of the box.

**C#** actually gives you two options: `SortedDictionary<K,V>` (Red-Black tree, best for frequent inserts/deletes) and `SortedList<K,V>` (sorted array, best for infrequent updates with fast indexed access).

### Why Rust Chose B-Tree Over Red-Black Tree

This is a deliberate, whole-system optimization. The difference comes down to **cache behavior**:

```text
  Red-Black Tree node:                BTreeMap node (B=6):
  ┌──────────────────────┐            ┌──────────────────────────────────────┐
  │ key │ value │ color  │            │ key₁ val₁ key₂ val₂ ... key₁₁ val₁₁│
  │ *left │ *right │ *parent │        │         contiguous in memory         │
  └──────────────────────┘            └──────────────────────────────────────┘
  1 key + 3 pointers                  Up to 11 keys in contiguous memory
  → cache miss on EVERY comparison    → ONE cache fill, MANY comparisons
```

A Red-Black tree comparison touches `1 key + 3 pointers` spread across heap-allocated nodes — that's a **cache miss on every step** down the tree. A BTreeMap node packs up to `11 keys` in contiguous memory — one cache fill loads many keys to compare against.

This is why benchmarks show Rust's `BTreeMap` often **outperforms** Red-Black tree implementations for sorted operations, despite both being O(log n) theoretically. Constants matter, and cache misses cost ~100 CPU cycles each.

```rust
use std::collections::BTreeMap;

fn main() {
    // Leaderboard (sorted by score, highest first)
    let mut leaderboard: BTreeMap<std::cmp::Reverse<i32>, String> = BTreeMap::new();
    
    use std::cmp::Reverse;
    leaderboard.insert(Reverse(100), "Alice".into());
    leaderboard.insert(Reverse(85), "Bob".into());
    leaderboard.insert(Reverse(92), "Carol".into());
    
    for (rank, (Reverse(score), name)) in leaderboard.iter().enumerate() {
        println!("#{}: {} ({})", rank + 1, name, score);
    }
    // #1: Alice (100)
    // #2: Carol (92)
    // #3: Bob (85)
}
```

---

## When to Use BTreeMap vs HashMap — Real Decisions

Let's be honest: **HashMap is faster for pure key-value lookups**. Typical benchmarks show ~30-80ns for `HashMap::get` vs ~200-400ns for `BTreeMap::get`. That's a 3-5x difference. So when should you reach for `BTreeMap`?

### The Decision Framework

```text
  Do you need sorted iteration?  ──── Yes ──→  BTreeMap
         │ No
  Do you need range queries?     ──── Yes ──→  BTreeMap
         │ No
  Do you need min/max key?       ──── Yes ──→  BTreeMap
         │ No
  Does your key implement Hash?  ──── No  ──→  BTreeMap
         │ Yes
  Need deterministic ordering?   ──── Yes ──→  BTreeMap
         │ No
         └──→  HashMap (default choice)
```

### Real-World Scenarios

**Scenario 1 — Leaderboard:** You want the top-10 players. `BTreeMap` gives you `iter().rev().take(10)` — instant sorted access. With `HashMap`, you'd sort every time.

**Scenario 2 — Time-series data:** "Show me all events between 2pm and 3pm." `BTreeMap::range(1400..1500)` handles this in O(log n + k) where k is the result count. `HashMap` would require scanning every entry.

**Scenario 3 — Database index:** If you're implementing a simple in-memory index, `BTreeMap` is literally what real databases use. You get ordered scans for free.

**Scenario 4 — Deterministic tests:** `HashMap` iteration order varies between runs (and even between compilations). If your test output depends on iteration order, `BTreeMap` guarantees reproducibility.

```rust
use std::collections::{BTreeMap, HashMap};

fn main() {
    // HashMap: iteration order is NOT deterministic
    let hmap: HashMap<&str, i32> = HashMap::from([
        ("z", 1), ("a", 2), ("m", 3),
    ]);
    // Could print in any order — different each run!
    
    // BTreeMap: always sorted, always deterministic
    let bmap: BTreeMap<&str, i32> = BTreeMap::from([
        ("z", 1), ("a", 2), ("m", 3),
    ]);
    for (k, v) in &bmap {
        print!("{}: {}  ", k, v);
    }
    // Always: a: 2  m: 3  z: 1
}
```

### The "Secret" Use Case: Deterministic Serialization

If you serialize a `HashMap` to JSON, the key order can change between runs. This breaks **content-addressable storage** (where you hash the serialized bytes to get a unique ID). `BTreeMap` guarantees the same key order every time → same serialized bytes → same hash. Libraries like `serde_json` preserve the map's iteration order.

### Performance Trap to Avoid

On **both** `HashMap` and `BTreeMap`, avoid the double-lookup pattern:

```rust
// BAD: two lookups
if !map.contains_key(&key) {
    map.insert(key, compute_value());
}

// GOOD: one lookup with entry API
map.entry(key).or_insert_with(|| compute_value());
```

### Rule of Thumb

> **Start with `HashMap`. Switch to `BTreeMap` when you need ordering.** Most programs deal with unordered key-value data. But the moment you need sorted iteration, range queries, min/max, or deterministic output — `BTreeMap` is the right tool.

---

## Exercises

### Exercise 1: Frequency Table

Given a list of words, print them in alphabetical order with their frequency:

<details>
<summary>Solution</summary>

```rust
use std::collections::BTreeMap;

fn frequency_table(words: &[&str]) {
    let mut freq: BTreeMap<&str, usize> = BTreeMap::new();
    for &word in words {
        *freq.entry(word).or_insert(0) += 1;
    }
    for (word, count) in &freq {
        println!("{}: {}", word, count);
    }
}

fn main() {
    frequency_table(&["banana", "apple", "cherry", "apple", "banana", "apple"]);
    // apple: 3
    // banana: 2
    // cherry: 1
}
```

</details>

### Exercise 2: Range Filter

Given a BTreeMap of student scores, return all students scoring between `low` and `high` inclusive:

<details>
<summary>Solution</summary>

```rust
use std::collections::BTreeMap;

fn students_in_range<'a>(
    scores: &'a BTreeMap<&str, i32>,
    low: i32,
    high: i32,
) -> Vec<(&'a str, i32)> {
    scores
        .iter()
        .filter(|(_, &score)| score >= low && score <= high)
        .map(|(&name, &score)| (name, score))
        .collect()
}

fn main() {
    let scores = BTreeMap::from([
        ("Alice", 92),
        ("Bob", 67),
        ("Carol", 85),
        ("Dave", 43),
        ("Eve", 78),
    ]);
    
    let passing = students_in_range(&scores, 70, 90);
    println!("{:?}", passing);
    // [("Carol", 85), ("Eve", 78)]
}
```

</details>

---

## Summary

| Feature | Detail |
|---------|--------|
| Structure | B-tree (self-balancing) |
| Key requirement | `Ord` (not `Hash`) |
| All operations | O(log n) |
| Iteration order | Sorted by key |
| Range queries | `map.range(a..b)` |
| Min/Max | `first_key_value()` / `last_key_value()` |
| Entry API | Same as HashMap |
| Best for | Sorted data, ranges, predictable perf |

---

**Previous:** [← HashMaps](./04-hashmaps.md) · **Next:** [Sets →](./06-sets.md)

<p align="center"><i>Tutorial 5 of 8 — Stage 5: Collections</i></p>

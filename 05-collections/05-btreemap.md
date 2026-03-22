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

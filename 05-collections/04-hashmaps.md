# HashMaps 🗺️

> **`HashMap<K, V>` stores key-value pairs with O(1) average lookup. It's Rust's primary associative container — like `dict` in Python, `Map` in JavaScript, or `HashMap` in Java.**

---

## Table of Contents

- [Creating HashMaps](#creating-hashmaps)
- [Inserting and Updating](#inserting-and-updating)
- [Accessing Values](#accessing-values)
- [The Entry API](#the-entry-api)
- [Removing Entries](#removing-entries)
- [Iterating](#iterating)
- [Ownership and HashMaps](#ownership-and-hashmaps)
- [How Hashing Works](#how-hashing-works)
- [Custom Hash Functions](#custom-hash-functions)
- [Common Patterns](#common-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Creating HashMaps

```rust
use std::collections::HashMap;

fn main() {
    // Empty
    let mut scores: HashMap<String, i32> = HashMap::new();
    
    // With capacity
    let mut scores: HashMap<String, i32> = HashMap::with_capacity(100);
    
    // From tuples
    let scores: HashMap<&str, i32> = HashMap::from([
        ("Alice", 100),
        ("Bob", 85),
        ("Carol", 92),
    ]);
    
    // From two iterators (zip + collect)
    let names = vec!["Alice", "Bob", "Carol"];
    let points = vec![100, 85, 92];
    let scores: HashMap<&str, i32> = names.into_iter().zip(points).collect();
    
    // From iterator of tuples
    let scores: HashMap<&str, i32> = vec![("Alice", 100), ("Bob", 85)]
        .into_iter()
        .collect();
}
```

---

## Inserting and Updating

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    
    // Insert
    scores.insert("Alice", 100);
    scores.insert("Bob", 85);
    
    // Overwrite (insert returns the OLD value)
    let old = scores.insert("Alice", 95);
    println!("Old Alice score: {:?}", old);  // Some(100)
    
    let old = scores.insert("Carol", 88);
    println!("Old Carol score: {:?}", old);  // None (didn't exist)
    
    println!("{:?}", scores);
    // {"Alice": 95, "Bob": 85, "Carol": 88}
}
```

---

## Accessing Values

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::from([
        ("Alice", 100),
        ("Bob", 85),
    ]);
    
    // get() returns Option<&V>
    match scores.get("Alice") {
        Some(&score) => println!("Alice: {}", score),
        None => println!("Alice not found"),
    }
    
    // get() with if let
    if let Some(&score) = scores.get("Bob") {
        println!("Bob: {}", score);
    }
    
    // Index with [] — panics if key missing
    let score = scores["Alice"];  // 100
    // let score = scores["Dave"];  // PANIC!
    
    // get_key_value() returns Option<(&K, &V)>
    if let Some((name, score)) = scores.get_key_value("Alice") {
        println!("{}: {}", name, score);
    }
    
    // contains_key()
    println!("Has Alice? {}", scores.contains_key("Alice"));  // true
    
    // len() and is_empty()
    println!("Count: {}", scores.len());       // 2
    println!("Empty: {}", scores.is_empty());  // false
}
```

---

## The Entry API

The entry API is the idiomatic way to insert-or-update:

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<&str, i32> = HashMap::new();
    
    // Insert only if key doesn't exist
    scores.entry("Alice").or_insert(100);
    scores.entry("Alice").or_insert(200);  // Does nothing, Alice already exists
    println!("Alice: {}", scores["Alice"]);  // 100
    
    // Insert with a function (only called if key missing)
    scores.entry("Bob").or_insert_with(|| expensive_computation());
    
    // Insert default and get mutable reference
    let count = scores.entry("Carol").or_insert(0);
    *count += 10;  // Carol = 10
    
    // Word counter (classic pattern)
    let text = "hello world hello rust hello world";
    let mut word_count: HashMap<&str, i32> = HashMap::new();
    
    for word in text.split_whitespace() {
        let count = word_count.entry(word).or_insert(0);
        *count += 1;
    }
    println!("{:?}", word_count);
    // {"hello": 3, "world": 2, "rust": 1}
    
    // or_default() — uses Default trait
    let mut counts: HashMap<&str, Vec<i32>> = HashMap::new();
    counts.entry("scores").or_default().push(100);
    counts.entry("scores").or_default().push(85);
}

fn expensive_computation() -> i32 {
    42
}
```

Entry API variants:

```rust
use std::collections::HashMap;

fn main() {
    let mut map: HashMap<&str, i32> = HashMap::new();
    
    // Modify existing or insert
    map.entry("key")
        .and_modify(|v| *v += 1)
        .or_insert(1);
    // First call: inserts 1
    
    map.entry("key")
        .and_modify(|v| *v += 1)
        .or_insert(1);
    // Second call: increments to 2
    
    println!("{}", map["key"]);  // 2
}
```

---

## Removing Entries

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::from([
        ("Alice", 100),
        ("Bob", 85),
        ("Carol", 92),
    ]);
    
    // Remove by key
    let removed = scores.remove("Bob");
    println!("{:?}", removed);  // Some(85)
    
    let removed = scores.remove("Dave");
    println!("{:?}", removed);  // None
    
    // Remove with key-value pair returned
    let removed = scores.remove_entry("Alice");
    println!("{:?}", removed);  // Some(("Alice", 100))
    
    // Retain elements matching a predicate
    let mut scores = HashMap::from([
        ("Alice", 100),
        ("Bob", 50),
        ("Carol", 92),
        ("Dave", 30),
    ]);
    scores.retain(|_name, &mut score| score >= 60);
    println!("{:?}", scores);  // Only Alice and Carol remain
    
    // Clear everything
    scores.clear();
}
```

---

## Iterating

```rust
use std::collections::HashMap;

fn main() {
    let scores = HashMap::from([
        ("Alice", 100),
        ("Bob", 85),
        ("Carol", 92),
    ]);
    
    // Iterate over key-value pairs
    for (name, score) in &scores {
        println!("{}: {}", name, score);
    }
    
    // Keys only
    for name in scores.keys() {
        println!("{}", name);
    }
    
    // Values only
    for score in scores.values() {
        println!("{}", score);
    }
    
    // Mutable iteration
    let mut scores = scores;
    for score in scores.values_mut() {
        *score += 10;  // Give everyone 10 bonus points
    }
    
    // into_iter (consumes the map)
    for (name, score) in scores {
        println!("{} got {}", name, score);
    }
    // scores is now moved
}
```

**Note:** HashMap does NOT guarantee iteration order. If you need ordered iteration, use `BTreeMap`.

---

## Ownership and HashMaps

```rust
use std::collections::HashMap;

fn main() {
    // Types that implement Copy are copied
    let mut map = HashMap::new();
    let x = 10;
    map.insert("key", x);
    println!("x is still: {}", x);  // ✅ i32 is Copy
    
    // Types that don't implement Copy are moved
    let mut map = HashMap::new();
    let name = String::from("Alice");
    map.insert(name, 100);
    // println!("{}", name);  // ❌ name was moved into the map
    
    // Use references to avoid moving
    let name = String::from("Alice");
    let mut map = HashMap::new();
    map.insert(&name, 100);  // Borrows name
    println!("{}", name);     // ✅ Still usable
    // But: map can't outlive name!
    
    // Clone if you need both
    let mut map = HashMap::new();
    let name = String::from("Alice");
    map.insert(name.clone(), 100);
    println!("{}", name);  // ✅ name is the original
}
```

---

## How Hashing Works

```
Key "Alice" → hash function → hash value (e.g., 0x7A3F...)
                                    │
                                    ▼
              ┌───────────────────────────────────┐
Buckets:      │ 0 │ 1 │ 2 │ ... │ i │ ... │ n-1 │
              └───────────────────────────────────┘
                              hash % n = i
                                    │
                                    ▼
                          ("Alice", 100) stored here
```

- Keys must implement `Hash` + `Eq`
- Default hasher: SipHash 1-3 (DOS-resistant, not the fastest)
- All primitive types, `String`, `&str` implement `Hash`
- **Floats (`f32`, `f64`) do NOT implement `Hash`** (because NaN != NaN)

---

## Custom Hash Functions

```rust
use std::collections::HashMap;
use std::hash::BuildHasherDefault;

// For better performance (not DOS-resistant):
// Add to Cargo.toml: rustc-hash = "1"
// use rustc_hash::FxHashMap;
// let map: FxHashMap<String, i32> = FxHashMap::default();

// Using a key type that implements Hash
#[derive(Hash, Eq, PartialEq, Debug)]
struct StudentId {
    year: u32,
    number: u32,
}

fn main() {
    let mut grades: HashMap<StudentId, char> = HashMap::new();
    
    grades.insert(StudentId { year: 2024, number: 42 }, 'A');
    grades.insert(StudentId { year: 2024, number: 17 }, 'B');
    
    let id = StudentId { year: 2024, number: 42 };
    println!("{:?} got {}", id, grades[&id]);
}
```

---

## Common Patterns

```rust
use std::collections::HashMap;

// Group by
fn group_by_length(words: &[&str]) -> HashMap<usize, Vec<&str>> {
    let mut groups: HashMap<usize, Vec<&str>> = HashMap::new();
    for &word in words {
        groups.entry(word.len()).or_default().push(word);
    }
    groups
}

// Invert a map
fn invert(map: &HashMap<&str, i32>) -> HashMap<i32, &str> {
    map.iter().map(|(&k, &v)| (v, k)).collect()
}

// Two-sum problem
fn two_sum(nums: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut seen: HashMap<i32, usize> = HashMap::new();
    for (i, &num) in nums.iter().enumerate() {
        let complement = target - num;
        if let Some(&j) = seen.get(&complement) {
            return Some((j, i));
        }
        seen.insert(num, i);
    }
    None
}

fn main() {
    let words = vec!["hi", "hello", "hey", "world", "rust"];
    let groups = group_by_length(&words);
    println!("{:?}", groups);
    
    let result = two_sum(&[2, 7, 11, 15], 9);
    println!("{:?}", result);  // Some((0, 1))
}
```

---

## Exercises

### Exercise 1: Department Roster

Build a program that lets you add employees to departments and list them:

```rust
use std::collections::HashMap;

fn main() {
    let mut departments: HashMap<String, Vec<String>> = HashMap::new();
    
    add_employee(&mut departments, "Engineering", "Alice");
    add_employee(&mut departments, "Engineering", "Bob");
    add_employee(&mut departments, "Sales", "Carol");
    
    list_department(&departments, "Engineering");
    // Engineering: Alice, Bob
}
```

<details>
<summary>Solution</summary>

```rust
use std::collections::HashMap;

fn add_employee(
    departments: &mut HashMap<String, Vec<String>>,
    dept: &str,
    name: &str,
) {
    departments
        .entry(dept.to_string())
        .or_default()
        .push(name.to_string());
}

fn list_department(departments: &HashMap<String, Vec<String>>, dept: &str) {
    match departments.get(dept) {
        Some(employees) => {
            let mut sorted = employees.clone();
            sorted.sort();
            println!("{}: {}", dept, sorted.join(", "));
        }
        None => println!("{}: (no employees)", dept),
    }
}

fn main() {
    let mut departments: HashMap<String, Vec<String>> = HashMap::new();
    
    add_employee(&mut departments, "Engineering", "Alice");
    add_employee(&mut departments, "Engineering", "Bob");
    add_employee(&mut departments, "Sales", "Carol");
    add_employee(&mut departments, "Engineering", "Dave");
    
    list_department(&departments, "Engineering");
    list_department(&departments, "Sales");
    list_department(&departments, "Marketing");
}
```

</details>

---

## Summary

| Operation | Method | Time |
|-----------|--------|------|
| Insert | `insert(k, v)` | O(1) avg |
| Lookup | `get(&k)` / `map[&k]` | O(1) avg |
| Remove | `remove(&k)` | O(1) avg |
| Contains | `contains_key(&k)` | O(1) avg |
| Entry API | `entry(k).or_insert(v)` | O(1) avg |
| Length | `len()` | O(1) |
| Iterate | `for (k,v) in &map` | O(n) |

**Requirements:** Key type must implement `Hash + Eq`.

---

**Previous:** [← String Internals](./03-string-internals.md) · **Next:** [BTreeMap →](./05-btreemap.md)

<p align="center"><i>Tutorial 4 of 8 — Stage 5: Collections</i></p>

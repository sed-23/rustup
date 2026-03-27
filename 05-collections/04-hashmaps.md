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

### Hash Tables Across Programming Languages

The hash table is arguably the **most important data structure in computer science**. Its O(1) average-case lookup, insertion, and deletion make it the backbone of everything from databases and caches to compilers and interpreters. Every major language has one — but the implementations differ wildly.

```
           The Universal Data Structure
           ============================

  Language    Type Name         What Happens Under the Hood
  ─────────   ──────────────    ────────────────────────────────────
  Python      dict              Open addressing + random probing
  JavaScript  Map / Object      V8 hidden classes / hash table
  Java        HashMap           Separate chaining → red-black trees
  C++         unordered_map     Separate chaining (cache-unfriendly)
  Go          map               Hash table with 8-entry buckets
  Rust        HashMap           SwissTable (hashbrown) + SipHash
```

**Python `dict`:** Since Python 3.6, dictionaries preserve insertion order. Internally, CPython uses open addressing with pseudo-random probing. The hash table is split into a dense array of entries and a sparse index table, which makes iteration fast and memory-efficient.

**JavaScript `Map`/`Object`:** V8 (the engine behind Node.js and Chrome) uses "hidden classes" to optimize plain objects — they behave more like structs than hash tables when keys are known at compile time. `Map` is a proper hash table for dynamic key-value storage.

**Java `HashMap`:** Uses separate chaining — each bucket is a linked list. Since Java 8, when a single chain exceeds 8 elements, it converts to a **red-black tree** to avoid O(n) worst-case lookups. Load factor threshold is 0.75.

**C++ `std::unordered_map`:** Also separate chaining. Notoriously **cache-unfriendly** because each bucket is a pointer-linked list scattered across memory. This is one area where C++ performs significantly worse than Rust.

**Go `map`:** A built-in type backed by a hash table with buckets holding 8 key-value pairs each. Overflow buckets are chained when a bucket fills up. Maps are not safe for concurrent use — you need `sync.Map` or a mutex.

**Rust `HashMap`:** Uses **SwissTable** (from the `hashbrown` crate) — one of the fastest hash table implementations in any language. Developed at Google for their Abseil C++ library, Rust adopted it in 2019 (Rust 1.36). SwissTable uses SIMD instructions to probe **16 slots simultaneously**, making it dramatically faster than traditional implementations.

| Language | Type | Strategy | Preserves Order? | Default Hash |
|----------|------|----------|------------------|--------------|
| Python | `dict` | Open addressing | Yes (3.6+) | SipHash 1-3 |
| JavaScript | `Map` | Hash table | Yes (spec) | V8 internal |
| Java | `HashMap` | Chaining → RB-tree | No (`LinkedHashMap` does) | `hashCode()` |
| C++ | `unordered_map` | Separate chaining | No | `std::hash` |
| Go | `map` | Buckets of 8 | No | AES-based |
| Rust | `HashMap` | SwissTable (flat) | No (`IndexMap` does) | SipHash 1-3 |

> **Key takeaway:** Rust's `HashMap` is backed by SwissTable — the same algorithm Google built for Abseil. It's one of the fastest general-purpose hash tables in production, often **2-3× faster** than C++ `unordered_map` for the same workloads.

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

### The Entry API — One of Rust's Most Elegant Designs

The "check if a key exists, then insert or update" pattern is incredibly common — and in most languages, it requires **two separate lookups**:

```
  The Two-Lookup Problem (every other language)
  ═════════════════════════════════════════════

  Step 1: LOOK UP the key  →  hash, find bucket, compare keys
  Step 2: INSERT/UPDATE     →  hash, find bucket, compare keys (AGAIN!)

  Rust's Entry API:
  ═════════════════
  Step 1: entry(key)        →  hash, find bucket, return a HANDLE
  Step 2: or_insert(val)    →  use the handle. No second lookup!
```

Here's how other languages solve (or don't solve) this problem:

**Python:**
```python
# Option 1: Two lookups
if key not in d:
    d[key] = default
# Option 2: setdefault (one call, but always creates the default)
d.setdefault(key, default)
# Option 3: defaultdict (changes the container type entirely)
from collections import defaultdict
d = defaultdict(int)
d[key] += 1
```

**Java (8+):**
```java
// computeIfAbsent — uses a lambda, but it's a method on the Map
map.computeIfAbsent(key, k -> expensiveComputation());
// merge — for combining values
map.merge(key, 1, Integer::sum);
```

**JavaScript / Go — no built-in solution:**
```javascript
// JavaScript: manual check every time
if (!map.has(key)) map.set(key, defaultValue);
```
```go
// Go: manual check every time
if _, ok := m[key]; !ok {
    m[key] = defaultValue
}
```

**Rust's entry API: ONE lookup, zero-cost abstraction:**
```rust
map.entry(key).or_insert(default);          // Simple default
map.entry(key).or_insert_with(|| f());      // Lazy — only computes if absent
map.entry(key).or_default();                // Uses Default trait
*map.entry(key).or_insert(0) += 1;          // The classic counter idiom
```

Under the hood, `entry()` returns an **enum**:

```rust
pub enum Entry<'a, K, V> {
    Occupied(OccupiedEntry<'a, K, V>),  // Key exists — here's a &mut V
    Vacant(VacantEntry<'a, K, V>),      // Key absent — ready to insert
}
```

You can match on it for complex logic:
```rust
use std::collections::hash_map::Entry;
match map.entry(key) {
    Entry::Occupied(mut e) => {
        *e.get_mut() += 1;            // Update existing
    }
    Entry::Vacant(e) => {
        e.insert(compute_initial());  // Insert new
    }
}
```

| Language | Idiom | Lookups | Lazy? |
|----------|-------|---------|-------|
| Python | `d.setdefault(k, v)` | 1 | No (always creates default) |
| Python | `defaultdict` | 1 | Yes (factory function) |
| Java | `computeIfAbsent(k, fn)` | 1 | Yes (lambda) |
| JavaScript | `if (!has) set` | 2 | N/A |
| Go | `if _, ok; !ok` | 2 | N/A |
| Rust | `entry(k).or_insert(v)` | **1** | **Yes** (`or_insert_with`) |

> **The origin story:** Rust's Entry API was designed *because* the borrow checker makes the naive "check then insert" pattern impossible — you'd need both an immutable borrow (to check) and a mutable borrow (to insert) simultaneously. Rather than working around this limitation, the Rust team turned it into the **most ergonomic conditional-insert API** in any mainstream language. A limitation became an elegant solution.

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

### How Hashing Actually Works — Inside SwissTable

Rust's default hasher is **SipHash 1-3** — a cryptographic-quality hash function that trades raw speed for **security against HashDoS attacks**.

**What is HashDoS?** An attacker crafts input keys that ALL hash to the same bucket. With separate chaining, every insertion and lookup degrades to O(n), turning your O(n) algorithm into O(n²). In 2011, researchers demonstrated HashDoS against PHP, Java, Python, and Ruby web frameworks — a single HTTP request with carefully crafted POST parameters could lock up a server for minutes.

```
  Normal Distribution              HashDoS Attack
  ════════════════════             ════════════════════
  Bucket 0: ●●                    Bucket 0:
  Bucket 1: ●●●                   Bucket 1:
  Bucket 2: ●                     Bucket 2: ●●●●●●●●●●●●●●●●
  Bucket 3: ●●                    Bucket 3:
  Bucket 4: ●●●●                  Bucket 4:
  → O(1) lookups                  → O(n) lookups = total O(n²)
```

Rust defaults to SipHash for safety. For performance-critical code where DoS isn't a concern (game engines, compilers, offline processing), you can switch to faster hashers:

- **FxHash** (`rustc-hash` crate): Used inside the Rust compiler itself. ~3-5× faster than SipHash
- **AHash** (`ahash` crate): Uses AES hardware instructions when available. Default in `hashbrown` directly
- **xxHash**, **wyhash**: Other fast alternatives for non-adversarial data

**SwissTable Internal Layout:**

```
  Traditional Hash Table:           SwissTable:
  ┌──────────────────────┐          ┌─────────────┐  ┌──────────────────────┐
  │ Bucket 0 → linked    │          │ Control     │  │ Slots (flat array)   │
  │ Bucket 1 → linked    │          │ Bytes (1B   │  │ [KV][KV][KV][KV]... │
  │ Bucket 2 → linked    │          │ per slot)   │  │                      │
  │ ...pointers everywhere│          │ [H2|H2|H2]  │  │ Cache-friendly!      │
  └──────────────────────┘          └─────────────┘  └──────────────────────┘
                                    ↑ SIMD scans 16 at once
```

SwissTable replaces the traditional bucket-and-chain approach with:
1. **A flat array of slots** — key-value pairs stored contiguously in memory (cache-friendly)
2. **A separate array of control bytes** — one byte per slot encoding the slot's state

Each control byte is either:
- `0xFF` — the slot is **empty**
- `0x80` — the slot was **deleted** (tombstone)
- Top 7 bits of the hash (called **H2**) — the slot is **occupied**

**The SIMD trick:** When looking up a key, SwissTable:
1. Hashes the key → uses the low bits to find a **group of 16 slots**
2. Loads 16 control bytes into a SIMD register (one CPU instruction)
3. Compares all 16 against the H2 hash value **simultaneously** (one instruction!)
4. Only checks actual key equality for the 0-2 slots that matched

This is why `hashbrown` (SwissTable) is **~2× faster** than traditional implementations — it probes 16 candidates in the time it takes others to probe 1.

| Property | Traditional (Java/C++) | SwissTable (Rust) |
|----------|----------------------|-------------------|
| Layout | Buckets + linked lists | Flat array + control bytes |
| Probing | One slot at a time | 16 slots via SIMD |
| Load factor | 75% (Java) | 87.5% (7/8) |
| Cache behavior | Pointer chasing | Linear, cache-friendly |
| Empty overhead | 8 bytes/bucket | 1 byte/slot |

> **Practical impact:** If you're coming from C++ `unordered_map`, Rust's `HashMap` is likely **2-3× faster** for the same operations — and it's the *default*, with zero extra configuration.

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

# Choosing the Right Collection 🧭

> **Rust provides many collection types. Picking the right one means understanding your access patterns, performance requirements, and data constraints. This guide gives you a decision framework and performance comparison.**

---

## Table of Contents

- [Collection Overview](#collection-overview)
- [Decision Flowchart](#decision-flowchart)
- [Performance Comparison](#performance-comparison)
- [Memory Overhead](#memory-overhead)
- [Real-World Scenarios](#real-world-scenarios)
- [Conversion Between Collections](#conversion-between-collections)
- [Tips and Best Practices](#tips-and-best-practices)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Collection Overview

| Collection | Description | Key Trait |
|------------|-------------|-----------|
| `Vec<T>` | Growable array | — |
| `VecDeque<T>` | Double-ended queue (ring buffer) | — |
| `LinkedList<T>` | Doubly-linked list | — |
| `HashMap<K,V>` | Hash table | `K: Hash + Eq` |
| `BTreeMap<K,V>` | B-tree sorted map | `K: Ord` |
| `HashSet<T>` | Hash-based set | `T: Hash + Eq` |
| `BTreeSet<T>` | B-tree sorted set | `T: Ord` |
| `BinaryHeap<T>` | Max-heap / priority queue | `T: Ord` |

---

## Decision Flowchart

```
What do you need?
│
├─ Sequence of elements (ordered by insertion)?
│  ├─ Need fast push/pop at BOTH ends? → VecDeque
│  ├─ Need priority ordering (max/min first)? → BinaryHeap
│  └─ Otherwise → Vec  (default choice)
│
├─ Key → Value mapping?
│  ├─ Need keys sorted / range queries? → BTreeMap
│  └─ Otherwise → HashMap  (default choice)
│
├─ Set of unique values?
│  ├─ Need sorted / range queries? → BTreeSet
│  └─ Otherwise → HashSet  (default choice)
│
└─ Special cases:
   ├─ O(1) concatenation of two lists → LinkedList (rare)
   ├─ Stack (LIFO) → Vec (push/pop)
   └─ Queue (FIFO) → VecDeque (push_back/pop_front)
```

---

## Performance Comparison

### Sequences

| Operation | `Vec` | `VecDeque` | `LinkedList` |
|-----------|-------|------------|--------------|
| Push back | O(1)* | O(1)* | O(1) |
| Push front | **O(n)** | O(1)* | O(1) |
| Pop back | O(1) | O(1) | O(1) |
| Pop front | **O(n)** | O(1) | O(1) |
| Index `[i]` | O(1) | O(1) | **O(n)** |
| Insert middle | O(n) | O(n) | O(1)† |
| Remove middle | O(n) | O(n) | O(1)† |
| Search | O(n) | O(n) | O(n) |
| Sort | O(n log n) | O(n log n) | O(n log n) |
| Concat two | O(n) | O(n) | **O(1)** |
| Cache perf | Excellent | Good | Poor |

\* amortized · † if you already have a cursor at the position

### Maps

| Operation | `HashMap` | `BTreeMap` |
|-----------|-----------|------------|
| Insert | O(1)* | O(log n) |
| Lookup | O(1)* | O(log n) |
| Remove | O(1)* | O(log n) |
| Min / Max key | O(n) | **O(log n)** |
| Range query | Not supported | **O(log n + k)** |
| Iteration order | Random | **Sorted** |
| Worst case | O(n) | O(log n) |

\* average case

### Sets

| Operation | `HashSet` | `BTreeSet` |
|-----------|-----------|------------|
| Insert | O(1)* | O(log n) |
| Contains | O(1)* | O(log n) |
| Union / Intersection | O(n + m) | O(n + m) |
| Min / Max | O(n) | **O(log n)** |
| Range query | Not supported | **O(log n + k)** |

### Priority Queue

| Operation | `BinaryHeap` | Sorted `Vec` | `BTreeSet` |
|-----------|-------------|-------------|------------|
| Push | O(log n) | O(n) | O(log n) |
| Pop max | O(log n) | O(1) | O(log n) |
| Peek max | O(1) | O(1) | O(log n) |
| Arbitrary remove | O(n) | O(n) | O(log n) |

---

## Memory Overhead

```
Vec<T>:        24 bytes stack + (cap × size_of::<T>()) heap
VecDeque<T>:   24 bytes stack + (cap × size_of::<T>()) heap  
LinkedList<T>: 24 bytes stack + (n × (size_of::<T>() + 16)) heap
HashMap<K,V>:  ~56 bytes stack + buckets + entries
BTreeMap<K,V>: ~24 bytes stack + tree nodes
HashSet<T>:    Same as HashMap<T, ()>
BTreeSet<T>:   Same as BTreeMap<T, ()>
BinaryHeap<T>: Same as Vec<T>
```

For `LinkedList`, each node has 2 pointers (prev + next) = 16 bytes overhead per element. For a list of `i32`, that's 16 bytes overhead for 4 bytes of data — 80% waste!

---

## Real-World Scenarios

### Scenario 1: Log Processing

```rust
// Reading log lines → Vec (sequential access)
let lines: Vec<String> = read_log_file();

// Counting error types → HashMap
let mut error_counts: HashMap<String, usize> = HashMap::new();
for line in &lines {
    if let Some(error_type) = extract_error(line) {
        *error_counts.entry(error_type).or_insert(0) += 1;
    }
}

// Top 10 errors → BinaryHeap
use std::collections::BinaryHeap;
let mut heap: BinaryHeap<(usize, String)> = error_counts
    .into_iter()
    .map(|(k, v)| (v, k))
    .collect();

for _ in 0..10 {
    if let Some((count, error)) = heap.pop() {
        println!("{}: {}", error, count);
    }
}
```

### Scenario 2: Web Server Request Queue

```rust
use std::collections::VecDeque;

// FIFO queue for requests
let mut request_queue: VecDeque<Request> = VecDeque::new();
request_queue.push_back(new_request);    // Enqueue
let next = request_queue.pop_front();    // Dequeue
```

### Scenario 3: Unique Visitors

```rust
use std::collections::HashSet;

// Track unique IPs
let mut unique_ips: HashSet<String> = HashSet::new();
for request in requests {
    unique_ips.insert(request.ip.clone());
}
println!("Unique visitors: {}", unique_ips.len());
```

### Scenario 4: Leaderboard

```rust
use std::collections::BTreeMap;

// Sorted scores → BTreeMap with Reverse key for descending order
use std::cmp::Reverse;
let mut leaderboard: BTreeMap<Reverse<u64>, String> = BTreeMap::new();
leaderboard.insert(Reverse(1500), "Alice".into());
leaderboard.insert(Reverse(1200), "Bob".into());

// Top 3 easily:
for (i, (Reverse(score), name)) in leaderboard.iter().take(3).enumerate() {
    println!("#{}: {} ({})", i + 1, name, score);
}
```

---

## Conversion Between Collections

```rust
use std::collections::*;

fn main() {
    // Vec ↔ VecDeque
    let v: Vec<i32> = vec![1, 2, 3];
    let d: VecDeque<i32> = VecDeque::from(v);
    let v: Vec<i32> = Vec::from(d);
    
    // Vec → HashSet (dedup)
    let v = vec![1, 2, 2, 3, 3, 3];
    let s: HashSet<i32> = v.into_iter().collect();
    
    // HashSet → Vec
    let v: Vec<i32> = s.into_iter().collect();
    
    // Vec → BinaryHeap
    let v = vec![3, 1, 4, 1, 5];
    let heap: BinaryHeap<i32> = v.into();
    
    // BinaryHeap → sorted Vec
    let sorted: Vec<i32> = heap.into_sorted_vec();
    
    // HashMap → Vec of tuples
    let map = HashMap::from([("a", 1), ("b", 2)]);
    let pairs: Vec<(&str, i32)> = map.into_iter().collect();
    
    // Vec of tuples → HashMap
    let pairs = vec![("a", 1), ("b", 2)];
    let map: HashMap<&str, i32> = pairs.into_iter().collect();
    
    // HashMap ↔ BTreeMap
    let hash_map = HashMap::from([("b", 2), ("a", 1)]);
    let btree: BTreeMap<&str, i32> = hash_map.into_iter().collect();
    let hash_map: HashMap<&str, i32> = btree.into_iter().collect();
}
```

---

## Tips and Best Practices

1. **Start with `Vec`** — it's the fastest for most workloads due to cache locality
2. **Pre-allocate** — use `with_capacity()` when you know the size
3. **Use `&[T]` in function params** — accepts both `Vec` and slices
4. **Prefer `HashMap` over `BTreeMap`** — unless you need sorting
5. **Use `entry()` API** — more efficient than `get` + `insert` separately
6. **`swap_remove()` for unordered `Vec`** — O(1) instead of O(n) `remove()`
7. **Consider `SmallVec`** (from crate) — stores small arrays on the stack
8. **`BinaryHeap` for top-k problems** — more efficient than sorting everything
9. **`BTreeSet` for sorted dedup** — simpler than sort + dedup on Vec

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    
    // BAD: O(n) remove
    v.remove(2);
    
    // GOOD: O(1) if order doesn't matter
    let mut v = vec![1, 2, 3, 4, 5];
    v.swap_remove(2);  // Now: [1, 2, 5, 4]
}
```

---

## Exercises

### Exercise 1: Choose the Collection

For each scenario, which collection would you use?

1. Storing a deck of cards you draw from the top
2. Finding the 5 most frequent words in a document
3. Checking if a username is already taken
4. Task scheduling where highest-priority runs first
5. Storing configuration key-value pairs for display in alphabetical order

<details>
<summary>Answers</summary>

1. **`Vec<Card>`** — push to back, pop from back (stack)
2. **`HashMap<String, usize>`** for counting + **`BinaryHeap`** for top-5
3. **`HashSet<String>`** — O(1) contains check
4. **`BinaryHeap<Task>`** — always pops highest priority
5. **`BTreeMap<String, String>`** — sorted iteration by key

</details>

---

## Summary

| Default | Alternative | Switch When |
|---------|-------------|-------------|
| `Vec` | `VecDeque` | Need fast front operations |
| `Vec` | `BinaryHeap` | Need priority ordering |
| `HashMap` | `BTreeMap` | Need sorted keys/ranges |
| `HashSet` | `BTreeSet` | Need sorted elements/ranges |
| Any | `LinkedList` | Almost never (O(1) split/join only) |

**Rule of thumb:** Start with `Vec`, `HashMap`, or `HashSet`. Switch only when you have a specific need.

---

**Previous:** [← VecDeque and Others](./07-vecdeque-and-others.md)

<p align="center"><i>Tutorial 8 of 8 — Stage 5: Collections</i></p>

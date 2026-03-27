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

### The 80/20 Rule of Collections

In real-world Rust codebases, the vast majority of collection usage boils down to just **two types**: `Vec` and `HashMap`. Understanding this distribution saves you from choice paralysis.

#### Usage Distribution in Practice

```
  Collection Usage in Real Rust Codebases (approximate)
  ┌──────────────────────────────────────────────────────────┐
  │ Vec<T>      ████████████████████████████████████░░  ~60% │
  │ HashMap     ████████████░░░░░░░░░░░░░░░░░░░░░░░░░  ~20% │
  │ String      ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~15% │
  │ Everything  ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   ~5% │
  │ else                                                     │
  └──────────────────────────────────────────────────────────┘
```

This pattern holds across the Rust standard library survey (2020) and aligns with other languages:

| Language | "The List" | "The Map" | Combined % |
|----------|-----------|----------|------------|
| Rust | `Vec` | `HashMap` | ~80% |
| Python | `list` | `dict` | ~80% |
| Java | `ArrayList` | `HashMap` | ~75% |
| Go | slice | `map` | ~85% |
| C++ | `std::vector` | `std::unordered_map` | ~70% |

#### Why Just Two?

Most real programs are doing one of two things:

1. **"Process a list of things"** — reading records, transforming data, filtering results → `Vec`
2. **"Look up things by key"** — caching, counting, indexing, deduplication → `HashMap`

#### The Learning Strategy

> **Master `Vec` and `HashMap` first.** They cover ~80% of your needs. The rest — `BTreeMap`, `VecDeque`, `BinaryHeap` — are specialized tools you learn when a specific problem demands them.

#### Beyond std: Popular Collection Crates

When the standard library isn't enough, the ecosystem has you covered:

| Crate | What It Is | Use When |
|-------|-----------|----------|
| `indexmap` | Insertion-ordered `HashMap` | You need map iteration in insertion order |
| `smallvec` | Stack-allocated small `Vec` | Most of your vecs hold ≤ N elements |
| `tinyvec` | Like `smallvec` but no `unsafe` | You want stack-small vecs without unsafe |
| `slab` | Pre-allocated arena with stable keys | You need O(1) insert/remove with integer keys |
| `dashmap` | Concurrent `HashMap` | You need shared map access across threads |
| `im` | Immutable/persistent collections | You want structural sharing (functional style) |

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

### Collection Performance — Beyond Big-O

Big-O notation is a useful mental model, but it hides **constant factors** — and those constant factors can be enormous. Understanding what happens beneath the abstraction is what separates good Rust engineers from great ones.

#### Cache Effects: The Hidden Dominator

Modern CPUs access memory through a hierarchy of caches. Contiguous memory (like `Vec`) stays in cache; scattered memory (like `LinkedList`) causes constant cache misses.

```
  Memory Access Latency (approximate)
  ┌────────────────────────────────────────────────────────┐
  │ L1 cache hit          ~1 ns     ████                   │
  │ L2 cache hit          ~4 ns     ████████████           │
  │ L3 cache hit         ~12 ns     ██████████████████████ │
  │ Main memory (RAM)    ~80 ns     ████████████████████████│
  │                                 ████████████████████████│
  │                                 ████████████████████████│
  └────────────────────────────────────────────────────────┘
```

| Operation | Vec | LinkedList | Both "O(n)" but... |
|-----------|-----|------------|---------------------|
| Iterate 1M elements | ~0.5 ns/elem | ~15 ns/elem | LinkedList is **30× slower** |
| Sequential sum | ~0.5 ms | ~15 ms | Same asymptotic complexity |
| Random access | ~1 ns | ~40 ns per hop | Not even close |

#### Allocation Cost

`Vec::push` (amortized) is faster than `LinkedList::push_back` because `Vec` doesn't allocate per element — it doubles its capacity and reuses the buffer. Each `LinkedList` node is a separate heap allocation.

```rust
// Vec: 1 allocation grows to hold many elements
let mut v = Vec::new();
for i in 0..1000 {
    v.push(i);  // ~10 reallocations total (doubling strategy)
}

// LinkedList: 1000 separate allocations!
let mut list = LinkedList::new();
for i in 0..1000 {
    list.push_back(i);  // 1000 malloc calls
}
```

#### Branch Prediction and BTreeMap

`BTreeMap` stores keys in sorted arrays within each node. The CPU's branch predictor can anticipate comparison outcomes in the sorted chain, keeping the pipeline full. `HashMap`'s hash function produces essentially random access patterns that the predictor can't help with.

#### The Crossover Point

For **small collections** (fewer than ~50 elements), a `Vec` with linear search often **beats** `HashMap`'s hashing overhead:

```
  Lookup Time vs Collection Size
  Time │
       │   HashMap ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
       │  /        ╱ Vec (linear search)
       │ /       ╱
       │/      ╱       ← crossover (~30-50 elements)
       │     ╱
       │   ╱
       │  ╱
       │╱
       └──────────────────────────────── Size
          10   30   50   100  500  1000
```

This is why some languages provide `SmallMap` types that use a `Vec` internally for small sizes and switch to a hash map when the size grows.

#### Benchmarking Guidance

```rust
// Use the `criterion` crate for reliable Rust benchmarks
// In Cargo.toml:
// [dev-dependencies]
// criterion = { version = "0.5", features = ["html_reports"] }

// Don't do this:
let start = std::time::Instant::now();
my_function();
println!("Took {:?}", start.elapsed());  // ← unreliable!

// Instead use criterion's statistical benchmarking
// which accounts for warmup, outliers, and noise.
```

> **General advice:** Profile before optimizing. The collection you *think* is slow might not be the bottleneck. Use `cargo flamegraph` or `perf` to find the real hotspot, then choose the right collection for that specific path.

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

### Collection Patterns in the Rust Ecosystem

Beyond choosing the right collection type, experienced Rust developers combine collections into **patterns** that solve recurring architectural problems. These patterns appear across web servers, game engines, compilers, and data pipelines.

#### Pattern 1: String Interning

When your program processes many duplicate strings, assign each unique string a numeric ID and work with integers instead:

```rust
use std::collections::HashMap;

struct StringInterner {
    map: HashMap<String, usize>,
    strings: Vec<String>,
}

impl StringInterner {
    fn intern(&mut self, s: &str) -> usize {
        if let Some(&id) = self.map.get(s) {
            return id;
        }
        let id = self.strings.len();
        self.strings.push(s.to_owned());
        self.map.insert(s.to_owned(), id);
        id
    }
    fn resolve(&self, id: usize) -> &str { &self.strings[id] }
}
```

> **Why?** Comparing two `usize` values is a single CPU instruction. Comparing two `String` values requires dereferencing pointers and scanning bytes. Compilers like `rustc` use this pattern extensively.

#### Pattern 2: Index Map (Decoupled Storage + Lookup)

Store data in a `Vec` for dense, cache-friendly iteration, and use a `HashMap` to index into it:

```rust
struct IndexMap<K: std::hash::Hash + Eq, V> {
    data: Vec<V>,
    index: HashMap<K, usize>,
}
// Insert: push to data, record index in map
// Lookup by key: map → index → data[index]
// Iterate: just iterate data (cache-friendly!)
```

This decouples **storage** (Vec, great for iteration) from **lookup** (HashMap, great for key access).

#### Pattern 3: ECS — Entity Component System

Game engines store components in parallel `Vec`s aligned by entity index:

```
  Entity IDs:       0       1       2       3
  ┌──────────────┬───────┬───────┬───────┬───────┐
  │ positions:   │ (0,0) │ (3,4) │ (1,2) │ (7,8) │  ← Vec<Position>
  │ velocities:  │ (1,0) │ (0,1) │ (2,2) │ (0,0) │  ← Vec<Velocity>
  │ healths:     │  100  │  80   │  None │  50   │  ← Vec<Option<Health>>
  └──────────────┴───────┴───────┴───────┴───────┘
```

Each system iterates a single `Vec` — maximum cache locality. Libraries like `bevy_ecs` and `hecs` are built on this idea.

#### Pattern 4: Adjacency List for Graphs

```rust
// Simple, cache-friendly graph representation
struct Graph {
    edges: Vec<Vec<usize>>,  // edges[node] = list of neighbors
}

impl Graph {
    fn new(num_nodes: usize) -> Self {
        Graph { edges: vec![vec![]; num_nodes] }
    }
    fn add_edge(&mut self, from: usize, to: usize) {
        self.edges[from].push(to);
    }
    fn neighbors(&self, node: usize) -> &[usize] {
        &self.edges[node]
    }
}
```

> Sparse graphs work well with `Vec<Vec<usize>>`. For dense graphs, consider a flat `Vec<bool>` adjacency matrix.

#### Pattern 5: Two-Phase Processing

Collect raw data first, then build optimized lookup structures:

```rust
// Phase 1: Collect
let mut raw: Vec<(String, u64)> = data_source.collect();

// Phase 2: Sort + dedup
raw.sort_by(|a, b| a.0.cmp(&b.0));
raw.dedup_by(|a, b| a.0 == b.0);

// Phase 3: Build lookup structure
let lookup: HashMap<String, u64> = raw.into_iter().collect();
```

This avoids the overhead of maintaining a sorted or hashed structure during the collection phase.

#### The `collect()` Superpower

Rust's `Iterator::collect()` can build **any** collection from an iterator, thanks to the `FromIterator` trait:

```rust
let input = vec![("alice", 90), ("bob", 85), ("carol", 92)];

// Collect into different collection types — same data!
let map: HashMap<&str, i32>  = input.iter().copied().collect();
let btree: BTreeMap<&str, i32> = input.iter().copied().collect();
let names: Vec<&str>          = input.iter().map(|(k, _)| *k).collect();
let unique: HashSet<&str>     = input.iter().map(|(k, _)| *k).collect();
let result: Result<HashMap<&str, i32>, _> = input
    .iter()
    .map(|&(k, v)| Ok((k, v)))
    .collect();  // Collects into Result<HashMap>!
```

`FromIterator` is one of Rust's most powerful traits — implementing it for your own types makes them work seamlessly with the iterator ecosystem.

> **Real-world insight:** Most performance issues with collections stem from **choosing the wrong collection** or **too many small allocations** — not from the collection's internal algorithms. Profile first, then pick the right pattern.

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

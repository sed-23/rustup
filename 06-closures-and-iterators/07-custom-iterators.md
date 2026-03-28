# Custom Iterators 🏗️

> **You can implement the `Iterator` trait for your own types to make them work seamlessly with `for` loops, iterator adaptors, and all the powerful methods in the standard library. This tutorial shows you how to build custom iterators from scratch.**

---

## Table of Contents

- [Implementing Iterator](#implementing-iterator)
- [A Simple Counter](#a-simple-counter)
- [Fibonacci Iterator](#fibonacci-iterator)
- [Iterating Over a Custom Collection](#iterating-over-a-custom-collection)
- [IntoIterator for Custom Types](#intoiterator-for-custom-types)
- [Borrowing vs Owning Iterators](#borrowing-vs-owning-iterators)
- [Double-Ended Iterators](#double-ended-iterators)
- [Iterator with Lifetime](#iterator-with-lifetime)
- [ExactSizeIterator](#exactsizeiterator)
- [Real-World Examples](#real-world-examples)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Implementing Iterator

To implement `Iterator`, you need:
1. A struct to hold the iterator state
2. Implement the `Iterator` trait with `type Item` and `fn next()`

```rust
pub trait Iterator {
    type Item;                                    // What we yield
    fn next(&mut self) -> Option<Self::Item>;     // Yield next, or None
}
```

---

## A Simple Counter

```rust
struct Counter {
    current: u32,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Self {
        Counter { current: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;
    
    fn next(&mut self) -> Option<u32> {
        if self.current < self.max {
            self.current += 1;
            Some(self.current)
        } else {
            None
        }
    }
}

fn main() {
    // Use in a for loop
    for n in Counter::new(5) {
        print!("{} ", n);
    }
    println!();
    // 1 2 3 4 5
    
    // All iterator methods automatically work!
    let sum: u32 = Counter::new(5).sum();
    println!("Sum: {}", sum);  // 15
    
    let evens: Vec<u32> = Counter::new(10)
        .filter(|x| x % 2 == 0)
        .collect();
    println!("{:?}", evens);  // [2, 4, 6, 8, 10]
    
    // Zip two counters
    let pairs: Vec<(u32, u32)> = Counter::new(5)
        .zip(Counter::new(5).rev()) // Won't work — need DoubleEndedIterator for rev
        .collect();
    
    // Zip with map
    let products: Vec<u32> = Counter::new(5)
        .zip(Counter::new(5).skip(1))
        .map(|(a, b)| a * b)
        .collect();
    println!("{:?}", products);  // [1*2, 2*3, 3*4, 4*5] = [2, 6, 12, 20]
}
```

---

### The Iterator Protocol Across Languages

Rust's `Iterator` trait is remarkably simple compared to iterator protocols in other languages. Understanding the differences highlights why Rust's design is both elegant and efficient.

**Python** uses `__iter__()` + `__next__()`. Termination is signaled by raising a `StopIteration` exception — yes, Python uses *exception handling* for normal control flow. Every single loop termination unwinds the call stack through the exception machinery, which is measurably slow in tight loops.

**Java** uses `Iterator<T>` with two methods: `hasNext()` and `next()`. This creates a two-phase protocol — you must check `hasNext()` before calling `next()`, and if you forget, you get a `NoSuchElementException` at runtime. Nothing in the type system prevents you from calling `next()` without checking first.

**C++** models iterators as *generalized pointers*. A container provides `begin()` and `end()`, and iterators support `++` (advance), `*` (dereference), and `==` (comparison). There are **five** iterator categories (Input, Output, Forward, Bidirectional, RandomAccess) each requiring different operator overloads — enormously complex to implement correctly.

**JavaScript** uses the `Symbol.iterator` protocol. The `next()` method returns an object `{ value, done }` — this means every single iteration step **allocates a new object on the heap**. In a tight loop over millions of elements, that's millions of tiny allocations for the garbage collector to clean up.

**Go** has **no standard iterator protocol** at all (prior to Go 1.23's range-over-func). Idiomatic Go uses channels (goroutines sending values, expensive) or callback functions. There's no unified way to compose iteration pipelines.

**Rust** requires exactly **one method**: `next(&mut self) -> Option<Self::Item>`. That's it. Termination is signaled by returning `None` — a zero-cost enum variant, not an exception. The compiler can optimize `Option<T>` to have zero overhead for many types (niche optimization). No heap allocation, no exceptions, no two-phase checking.

**Why `Option<T>` beats every alternative:**
- vs exceptions (Python): no stack unwinding, no runtime cost for termination
- vs boolean checks (Java): impossible to forget the check — you *must* pattern-match or unwrap
- vs sentinel values (C): type-safe, cannot confuse a valid value with "end"
- vs object allocation (JS): zero-cost, `None` is just a tag on the stack

| Language | Protocol | Methods to Implement | Termination Signal | Overhead Per Element |
|------------|------------------------------|----------------------|-------------------------------|------------------------------|
| Rust | `Iterator` trait | 1 (`next`) | `None` (zero-cost enum) | Zero — fully inlined |
| Python | `__iter__` + `__next__` | 2 | `StopIteration` exception | Exception machinery on end |
| Java | `Iterator<T>` | 2 (`hasNext`+`next`) | `false` from `hasNext()` | Boolean check + boxing |
| C++ | Pointer-like operators | 3–5 (`++`,`*`,`==`…) | Comparison with `end()` | Near-zero (but complex API) |
| JavaScript | `Symbol.iterator` | 1 (`next`) | `{ done: true }` | Object allocation per step |
| Go | (none standard) | N/A | Channel close / callback end | Goroutine + channel overhead |

Rust achieves the **simplest protocol** of any systems language while simultaneously being the **most efficient**. One method, zero allocation, full type safety.

---

## Fibonacci Iterator

```rust
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci { a: 0, b: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;
    
    fn next(&mut self) -> Option<u64> {
        let value = self.a;
        let next = self.a.checked_add(self.b)?;  // None on overflow
        self.a = self.b;
        self.b = next;
        Some(value)
    }
}

fn main() {
    // First 10 Fibonacci numbers
    let fibs: Vec<u64> = Fibonacci::new().take(10).collect();
    println!("{:?}", fibs);
    // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
    
    // Fibonacci numbers under 100
    let under_100: Vec<u64> = Fibonacci::new()
        .take_while(|&x| x < 100)
        .collect();
    println!("{:?}", under_100);
    // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
    
    // Sum of even Fibonacci numbers under 4 million
    let sum: u64 = Fibonacci::new()
        .take_while(|&x| x < 4_000_000)
        .filter(|x| x % 2 == 0)
        .sum();
    println!("Sum: {}", sum);
}
```

---

## Iterating Over a Custom Collection

```rust
struct Grid {
    data: Vec<Vec<i32>>,
    rows: usize,
    cols: usize,
}

impl Grid {
    fn new(rows: usize, cols: usize) -> Self {
        Grid {
            data: vec![vec![0; cols]; rows],
            rows,
            cols,
        }
    }
    
    fn set(&mut self, row: usize, col: usize, val: i32) {
        self.data[row][col] = val;
    }
    
    fn iter(&self) -> GridIter {
        GridIter {
            grid: self,
            row: 0,
            col: 0,
        }
    }
}

struct GridIter<'a> {
    grid: &'a Grid,
    row: usize,
    col: usize,
}

impl<'a> Iterator for GridIter<'a> {
    type Item = (usize, usize, &'a i32);
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.row >= self.grid.rows {
            return None;
        }
        
        let item = (self.row, self.col, &self.grid.data[self.row][self.col]);
        
        // Advance position
        self.col += 1;
        if self.col >= self.grid.cols {
            self.col = 0;
            self.row += 1;
        }
        
        Some(item)
    }
}

fn main() {
    let mut grid = Grid::new(2, 3);
    grid.set(0, 0, 1);
    grid.set(0, 1, 2);
    grid.set(0, 2, 3);
    grid.set(1, 0, 4);
    grid.set(1, 1, 5);
    grid.set(1, 2, 6);
    
    for (row, col, val) in grid.iter() {
        println!("[{},{}] = {}", row, col, val);
    }
    
    // Use iterator methods
    let sum: i32 = grid.iter().map(|(_, _, &v)| v).sum();
    println!("Sum: {}", sum);  // 21
}
```

---

## IntoIterator for Custom Types

Implement `IntoIterator` to make your type work with `for`:

```rust
struct Playlist {
    songs: Vec<String>,
}

impl Playlist {
    fn new() -> Self {
        Playlist { songs: Vec::new() }
    }
    
    fn add(&mut self, song: &str) {
        self.songs.push(song.to_string());
    }
}

// Consuming iterator (for song in playlist)
impl IntoIterator for Playlist {
    type Item = String;
    type IntoIter = std::vec::IntoIter<String>;
    
    fn into_iter(self) -> Self::IntoIter {
        self.songs.into_iter()
    }
}

// Borrowing iterator (for song in &playlist)
impl<'a> IntoIterator for &'a Playlist {
    type Item = &'a String;
    type IntoIter = std::slice::Iter<'a, String>;
    
    fn into_iter(self) -> Self::IntoIter {
        self.songs.iter()
    }
}

fn main() {
    let mut playlist = Playlist::new();
    playlist.add("Song A");
    playlist.add("Song B");
    playlist.add("Song C");
    
    // Borrow iteration
    for song in &playlist {
        println!("Playing: {}", song);
    }
    
    // Consuming iteration
    for song in playlist {
        println!("Moving: {}", song);
    }
    // playlist is consumed
}
```

---

## Borrowing vs Owning Iterators

A complete collection typically provides three iterators:

```rust
struct Stack<T> {
    data: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self { Stack { data: Vec::new() } }
    fn push(&mut self, val: T) { self.data.push(val); }
    
    // Borrowing iterator
    fn iter(&self) -> std::slice::Iter<T> {
        self.data.iter()
    }
    
    // Mutable borrowing iterator
    fn iter_mut(&mut self) -> std::slice::IterMut<T> {
        self.data.iter_mut()
    }
}

// Consuming iterator
impl<T> IntoIterator for Stack<T> {
    type Item = T;
    type IntoIter = std::vec::IntoIter<T>;
    
    fn into_iter(self) -> Self::IntoIter {
        self.data.into_iter()
    }
}

fn main() {
    let mut stack = Stack::new();
    stack.push(1);
    stack.push(2);
    stack.push(3);
    
    // Borrow
    for val in stack.iter() {
        print!("{} ", val);
    }
    println!();
    
    // Mutate
    for val in stack.iter_mut() {
        *val *= 10;
    }
    
    // Consume
    for val in stack {
        print!("{} ", val);
    }
    println!();
    // 10 20 30
}
```

---

### The Iterator Trait's 75+ Free Methods — What You Get for Free

When you implement a single method — `next()` — for your custom iterator, you instantly unlock the **entire** standard library iterator API. This is one of the most powerful payoffs in all of Rust's trait system.

Here is a (non-exhaustive) sampling of what you get for free:

| Category | Methods |
|---|---|
| Transforming | `map`, `flat_map`, `flatten`, `inspect`, `scan`, `enumerate` |
| Filtering | `filter`, `filter_map`, `skip`, `take`, `skip_while`, `take_while`, `step_by` |
| Combining | `zip`, `chain`, `interleave` (via itertools), `unzip` |
| Consuming | `collect`, `sum`, `product`, `fold`, `reduce`, `for_each`, `count` |
| Searching | `find`, `find_map`, `position`, `rposition`, `any`, `all` |
| Min/Max | `min`, `max`, `min_by`, `max_by`, `min_by_key`, `max_by_key` |
| Cloning | `cloned`, `copied` |
| Ordering | `cmp`, `partial_cmp`, `eq`, `ne`, `lt`, `le`, `gt`, `ge` |
| Peeking/Buffering | `peekable`, `fuse` |
| Reversing | `rev` (requires `DoubleEndedIterator`) |
| Size-aware | `last`, `nth`, `size_hint` |

That's roughly **75+ provided methods** with default implementations, all built on top of your single `next()` function.

**How does this work internally?** Each default method is generic over `Self` and simply calls `self.next()` in a loop. For example, here's a simplified version of how `fold` works:

```rust
// Inside the Iterator trait (simplified)
fn fold<B, F>(mut self, init: B, mut f: F) -> B
where
    F: FnMut(B, Self::Item) -> B,
{
    let mut acc = init;
    while let Some(item) = self.next() {
        acc = f(acc, item);
    }
    acc
}
```

Every other method — `sum`, `count`, `map`, `filter` — follows the same pattern: call `next()` repeatedly, apply some logic, return the result.

**Compare this with other languages:**
- **Java**: implement `Iterator<T>` and you get exactly **one** optional default method — `remove()`. That's it. For anything like `map` or `filter`, you need to convert to a `Stream`.
- **C++**: implementing an iterator gives you zero free methods. Each algorithm (`std::find`, `std::transform`, `std::accumulate`) is a standalone function that requires the correct iterator category.
- **Python**: implementing `__next__` gives you zero extra methods. You must wrap in `map()`, `filter()`, etc. as standalone builtins, or use list comprehensions.

**The `size_hint()` optimization:** Beyond `next()`, you can optionally override `size_hint()` to return `(lower_bound, Option<upper_bound>)`. This tells consumers like `collect()` how much memory to pre-allocate:

```rust
// Without size_hint: collect() starts with an empty Vec,
// reallocates 4 → 8 → 16 → 32 → ... as elements arrive.

// With accurate size_hint: collect() calls Vec::with_capacity(n)
// and does ZERO reallocations.
fn size_hint(&self) -> (usize, Option<usize>) {
    let remaining = (self.max - self.current) as usize;
    (remaining, Some(remaining))  // exact bounds
}
```

**`ExactSizeIterator`:** When your `size_hint()` returns exact bounds (lower == upper), you can also implement `ExactSizeIterator` to provide a `len()` method. This unlocks further optimizations — for example, `collect::<Vec<_>>()` can allocate exactly the right capacity with zero wasted space.

This design is a masterclass in trait-based API design: **minimal requirement** (one method), **maximum payoff** (75+ methods, zero-cost abstractions, composable pipelines). Every custom iterator you write instantly becomes a first-class citizen of Rust's iterator ecosystem.

---

## Double-Ended Iterators

Implement `DoubleEndedIterator` to support `.rev()` and `next_back()`:

```rust
struct Range {
    start: i32,
    end: i32,
}

impl Range {
    fn new(start: i32, end: i32) -> Self {
        Range { start, end }
    }
}

impl Iterator for Range {
    type Item = i32;
    
    fn next(&mut self) -> Option<i32> {
        if self.start < self.end {
            let val = self.start;
            self.start += 1;
            Some(val)
        } else {
            None
        }
    }
}

impl DoubleEndedIterator for Range {
    fn next_back(&mut self) -> Option<i32> {
        if self.start < self.end {
            self.end -= 1;
            Some(self.end)
        } else {
            None
        }
    }
}

fn main() {
    // Forward
    let forward: Vec<i32> = Range::new(1, 6).collect();
    println!("{:?}", forward);  // [1, 2, 3, 4, 5]
    
    // Reverse
    let backward: Vec<i32> = Range::new(1, 6).rev().collect();
    println!("{:?}", backward);  // [5, 4, 3, 2, 1]
    
    // From both ends
    let mut r = Range::new(1, 6);
    println!("{:?}", r.next());       // Some(1)
    println!("{:?}", r.next_back());  // Some(5)
    println!("{:?}", r.next());       // Some(2)
    println!("{:?}", r.next_back());  // Some(4)
    println!("{:?}", r.next());       // Some(3)
    println!("{:?}", r.next());       // None
}
```

---

## Iterator with Lifetime

When your iterator borrows from another structure:

```rust
struct Words<'a> {
    text: &'a str,
    pos: usize,
}

impl<'a> Words<'a> {
    fn new(text: &'a str) -> Self {
        Words { text, pos: 0 }
    }
}

impl<'a> Iterator for Words<'a> {
    type Item = &'a str;
    
    fn next(&mut self) -> Option<&'a str> {
        // Skip whitespace
        let start = self.text[self.pos..].find(|c: char| !c.is_whitespace())?;
        let start = self.pos + start;
        
        // Find end of word
        let end = self.text[start..]
            .find(|c: char| c.is_whitespace())
            .map(|i| start + i)
            .unwrap_or(self.text.len());
        
        self.pos = end;
        Some(&self.text[start..end])
    }
}

fn main() {
    let text = "  hello   beautiful   world  ";
    let words: Vec<&str> = Words::new(text).collect();
    println!("{:?}", words);  // ["hello", "beautiful", "world"]
}
```

---

## ExactSizeIterator

If you know the exact remaining length, implement this for optimizations:

```rust
struct Counter {
    current: u32,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Self {
        Counter { current: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;
    
    fn next(&mut self) -> Option<u32> {
        if self.current < self.max {
            self.current += 1;
            Some(self.current)
        } else {
            None
        }
    }
    
    // size_hint helps collect() pre-allocate
    fn size_hint(&self) -> (usize, Option<usize>) {
        let remaining = (self.max - self.current) as usize;
        (remaining, Some(remaining))
    }
}

impl ExactSizeIterator for Counter {
    fn len(&self) -> usize {
        (self.max - self.current) as usize
    }
}

fn main() {
    let counter = Counter::new(10);
    println!("Remaining: {}", counter.len());  // 10
    
    // collect() will pre-allocate exactly 10 elements
    let v: Vec<u32> = counter.collect();
    println!("{:?}", v);
}
```

---

## Real-World Examples

### CSV Row Iterator

```rust
struct CsvRows<'a> {
    lines: std::str::Lines<'a>,
}

impl<'a> CsvRows<'a> {
    fn new(data: &'a str) -> Self {
        CsvRows { lines: data.lines() }
    }
}

impl<'a> Iterator for CsvRows<'a> {
    type Item = Vec<&'a str>;
    
    fn next(&mut self) -> Option<Vec<&'a str>> {
        let line = self.lines.next()?;
        Some(line.split(',').map(|s| s.trim()).collect())
    }
}

fn main() {
    let csv = "name,age,city\nAlice,30,NYC\nBob,25,LA";
    
    for row in CsvRows::new(csv) {
        println!("{:?}", row);
    }
    // ["name", "age", "city"]
    // ["Alice", "30", "NYC"]
    // ["Bob", "25", "LA"]
}
```

### Chunk Iterator

```rust
struct Chunks<'a, T> {
    data: &'a [T],
    size: usize,
}

impl<'a, T> Chunks<'a, T> {
    fn new(data: &'a [T], size: usize) -> Self {
        assert!(size > 0);
        Chunks { data, size }
    }
}

impl<'a, T> Iterator for Chunks<'a, T> {
    type Item = &'a [T];
    
    fn next(&mut self) -> Option<&'a [T]> {
        if self.data.is_empty() {
            return None;
        }
        let chunk_size = self.size.min(self.data.len());
        let (chunk, rest) = self.data.split_at(chunk_size);
        self.data = rest;
        Some(chunk)
    }
}

fn main() {
    let data = [1, 2, 3, 4, 5, 6, 7];
    for chunk in Chunks::new(&data, 3) {
        println!("{:?}", chunk);
    }
    // [1, 2, 3]
    // [4, 5, 6]
    // [7]
}
```

---

### Real-World Custom Iterators — Parser, Tokenizer, Event Stream

Custom iterators aren't just a tutorial exercise — they're a foundational pattern used extensively across the Rust ecosystem. Here's how real crates leverage them:

**Production crates that rely on custom iterators:**
- **`regex`**: `Matches<'a>` lazily iterates over all regex matches in a string — only does work when you call `next()`
- **`serde_json`**: `StreamDeserializer` iterates over a stream of JSON values, parsing each one on demand
- **`walkdir`**: `WalkDir` recursively traverses directory trees lazily — it holds a stack of directories internally and yields entries one at a time, using constant memory regardless of tree depth
- **`csv`**: `Reader::records()` returns an iterator over CSV rows — can process **gigabytes** of CSV data with constant memory because it only holds one row in memory at a time

**Common patterns for custom iterators:**

**1. Stateful iterator** — the struct holds a cursor, buffer, or internal state; `next()` advances the state and yields the next item. The `GridIter` and `Words` examples above follow this pattern.

**2. Chunked processing** — the iterator yields fixed-size windows or chunks from a large input. This is how audio processing, network packet parsing, and batch database operations work.

**3. Infinite iterator** — like `Fibonacci`, these never return `None` on their own. They pair naturally with `.take(n)`, `.take_while(predicate)`, or `.zip()` with a finite iterator to produce finite output.

**4. Tree/graph traversal** — DFS or BFS iterators that hold a stack or queue internally. Compilers iterate over AST nodes, game engines iterate over scene graphs, and file managers iterate over directory trees — all using this pattern.

**Mini example — a Tokenizer iterator:**

```rust
/// Yields whitespace-delimited tokens from an input string.
struct Tokenizer<'a> {
    remaining: &'a str,
}

impl<'a> Tokenizer<'a> {
    fn new(input: &'a str) -> Self {
        Tokenizer { remaining: input }
    }
}

impl<'a> Iterator for Tokenizer<'a> {
    type Item = &'a str;

    fn next(&mut self) -> Option<&'a str> {
        // Skip leading whitespace
        self.remaining = self.remaining.trim_start();
        if self.remaining.is_empty() {
            return None;
        }

        // Find the end of the current token
        let end = self.remaining
            .find(char::is_whitespace)
            .unwrap_or(self.remaining.len());

        let token = &self.remaining[..end];
        self.remaining = &self.remaining[end..];
        Some(token)
    }
}

fn main() {
    let input = "  fn main()  { println!(42); }  ";

    // Instant access to the full iterator API:
    let tokens: Vec<&str> = Tokenizer::new(input).collect();
    println!("{:?}", tokens);
    // ["fn", "main()", "{", "println!(42);", "}"]

    // Count tokens
    let count = Tokenizer::new(input).count();
    println!("Token count: {}", count);  // 5

    // Find a specific token
    let has_fn = Tokenizer::new(input).any(|t| t == "fn");
    println!("Contains 'fn': {}", has_fn);  // true

    // Uppercase all tokens
    let upper: Vec<String> = Tokenizer::new(input)
        .map(|t| t.to_uppercase())
        .collect();
    println!("{:?}", upper);
    // ["FN", "MAIN()", "{", "PRINTLN!(42);", "}"]
}
```

**Key insight:** The `Tokenizer` struct implements only `next()` — a single method. Yet it instantly gains `collect`, `count`, `any`, `map`, `filter`, `fold`, `enumerate`, `zip`, and every other iterator method. Your custom iterator is a **first-class participant** in Rust's entire iterator ecosystem, composable with every adaptor and consumer in the standard library.

This is the power of Rust's trait system: define the minimum contract, and the ecosystem does the rest.

---

## Exercises

### Exercise 1: Cycle Iterator

Implement an iterator that cycles through a slice infinitely:

```rust
struct Cycle<'a, T> {
    data: &'a [T],
    index: usize,
}
```

<details>
<summary>Solution</summary>

```rust
struct Cycle<'a, T> {
    data: &'a [T],
    index: usize,
}

impl<'a, T> Cycle<'a, T> {
    fn new(data: &'a [T]) -> Self {
        Cycle { data, index: 0 }
    }
}

impl<'a, T> Iterator for Cycle<'a, T> {
    type Item = &'a T;
    
    fn next(&mut self) -> Option<&'a T> {
        if self.data.is_empty() {
            return None;
        }
        let item = &self.data[self.index % self.data.len()];
        self.index += 1;
        Some(item)
    }
}

fn main() {
    let colors = ["red", "green", "blue"];
    let pattern: Vec<&&str> = Cycle::new(&colors).take(7).collect();
    println!("{:?}", pattern);
    // ["red", "green", "blue", "red", "green", "blue", "red"]
}
```

</details>

### Exercise 2: StepBy Iterator

Implement a step iterator that yields every nth element:

<details>
<summary>Solution</summary>

```rust
struct StepBy<I> {
    iter: I,
    step: usize,
    first: bool,
}

impl<I: Iterator> StepBy<I> {
    fn new(iter: I, step: usize) -> Self {
        assert!(step > 0);
        StepBy { iter, step, first: true }
    }
}

impl<I: Iterator> Iterator for StepBy<I> {
    type Item = I::Item;
    
    fn next(&mut self) -> Option<I::Item> {
        if self.first {
            self.first = false;
            self.iter.next()
        } else {
            // Skip (step - 1) elements
            for _ in 0..self.step - 1 {
                self.iter.next()?;
            }
            self.iter.next()
        }
    }
}

fn main() {
    let v = vec![0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
    let stepped: Vec<&i32> = StepBy::new(v.iter(), 3).collect();
    println!("{:?}", stepped);  // [0, 3, 6, 9]
}
```

</details>

---

## Summary

| What | How |
|------|-----|
| Basic iterator | Implement `Iterator` with `type Item` and `next()` |
| Work with `for` loops | Implement `IntoIterator` for your type |
| Reverse iteration | Implement `DoubleEndedIterator` with `next_back()` |
| Exact size | Implement `ExactSizeIterator` with `len()` |
| Size hint | Override `size_hint()` for pre-allocation |
| Borrow from source | Use lifetime `'a` in iterator struct and `Item` |
| Three iterator types | `iter()` → `&T`, `iter_mut()` → `&mut T`, `into_iter()` → `T` |

Once you implement `Iterator`, you automatically get 70+ methods: `map`, `filter`, `fold`, `sum`, `collect`, and everything else.

---

**Previous:** [← Iterator Consumers](./06-iterator-consumers.md)

<p align="center"><i>Tutorial 7 of 7 — Stage 6: Closures and Iterators</i></p>

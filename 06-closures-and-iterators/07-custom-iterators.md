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

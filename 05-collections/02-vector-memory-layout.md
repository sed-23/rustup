# Vector Memory Layout 🧠

> **Understanding how `Vec<T>` stores data on the heap — capacity vs. length, reallocation strategy, and amortized cost — helps you write faster Rust code and avoid unnecessary allocations.**

---

## Table of Contents

- [The Three Fields of Vec](#the-three-fields-of-vec)
- [Stack vs Heap](#stack-vs-heap)
- [Capacity vs Length](#capacity-vs-length)
- [How Reallocation Works](#how-reallocation-works)
- [Growth Strategy](#growth-strategy)
- [Amortized Cost](#amortized-cost)
- [Avoiding Reallocations](#avoiding-reallocations)
- [Shrinking](#shrinking)
- [Zero-Sized Types](#zero-sized-types)
- [Memory Layout Diagram](#memory-layout-diagram)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Three Fields of Vec

Every `Vec<T>` contains exactly three machine-word-sized fields on the **stack**:

```
Vec<T> on the stack (24 bytes on 64-bit):
┌──────────────────┐
│ pointer (8 bytes) │ ──→ heap allocation
│ length  (8 bytes) │     number of elements currently stored
│ capacity(8 bytes) │     number of elements that CAN fit
└──────────────────┘
```

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    println!("Length:   {}", v.len());       // 3
    println!("Capacity: {}", v.capacity());  // 3 (or more)
    println!("Size of Vec on stack: {}", std::mem::size_of::<Vec<i32>>());  // 24
}
```

---

## Stack vs Heap

```
Stack                           Heap
┌────────────────────┐         ┌───┬───┬───┬───┬───┬───┬───┬───┐
│ ptr ─────────────────────→   │ 1 │ 2 │ 3 │   │   │   │   │   │
│ len: 3             │         └───┴───┴───┴───┴───┴───┴───┴───┘
│ cap: 8             │          ←─ len ─→ ←── unused capacity ──→
└────────────────────┘
```

- The **stack** holds the `Vec` struct (pointer + length + capacity)
- The **heap** holds the actual element data
- When `Vec` is dropped, the heap allocation is freed

---

## Capacity vs Length

```rust
fn main() {
    let mut v = Vec::with_capacity(10);
    
    // Nothing stored yet, but room for 10
    println!("len: {}, cap: {}", v.len(), v.capacity());
    // len: 0, cap: 10
    
    v.push(1);
    v.push(2);
    v.push(3);
    
    // 3 stored, room for 10
    println!("len: {}, cap: {}", v.len(), v.capacity());
    // len: 3, cap: 10
    
    // Access only up to len, not cap
    // v[5] would panic — even though capacity is 10
}
```

| Property | Meaning |
|----------|---------|
| `len()` | Number of elements actually stored |
| `capacity()` | Number of elements the buffer can hold before reallocation |
| Always | `len <= capacity` |

---

## How Reallocation Works

When you `push` and `len == capacity`, the vector must grow:

```
BEFORE push (len=4, cap=4):
Stack: ptr → Heap: [10, 20, 30, 40]
                    ↑ full!

STEP 1: Allocate new, larger buffer
         New heap: [__, __, __, __, __, __, __, __]  (cap=8)

STEP 2: Copy old data to new buffer
         New heap: [10, 20, 30, 40, __, __, __, __]

STEP 3: Free old buffer

STEP 4: Push new element
         New heap: [10, 20, 30, 40, 50, __, __, __]

AFTER (len=5, cap=8):
Stack: ptr → Heap: [10, 20, 30, 40, 50, __, __, __]
```

**Key consequence:** All pointers/references to elements are invalidated after reallocation. That's why Rust prevents holding a reference while pushing:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];     // Borrows v
    // v.push(4);           // ❌ Can't mutate while borrowed
    //                      // push might reallocate, making `first` a dangling pointer
    println!("{}", first);  // Use the borrow
    v.push(4);              // ✅ OK after borrow is done
}
```

---

## Growth Strategy

Rust's `Vec` doubles capacity (approximately) each time it needs to grow:

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    let mut last_cap = v.capacity();
    
    for i in 0..100 {
        v.push(i);
        if v.capacity() != last_cap {
            println!(
                "Grew at len={}: cap {} → {}",
                v.len(),
                last_cap,
                v.capacity()
            );
            last_cap = v.capacity();
        }
    }
}
```

Typical output:
```
Grew at len=1:  cap 0 → 4
Grew at len=5:  cap 4 → 8
Grew at len=9:  cap 8 → 16
Grew at len=17: cap 16 → 32
Grew at len=33: cap 32 → 64
Grew at len=65: cap 64 → 128
```

---

## Amortized Cost

Even though a single reallocation is O(n), **amortized** cost per push is O(1):

```
Push #  | Action           | Cost (elements copied)
1       | Allocate cap=4   | 0
2       | Push             | 0
3       | Push             | 0
4       | Push             | 0
5       | Realloc cap=8    | 4 (copy 4 old elements)
6       | Push             | 0
7       | Push             | 0
8       | Push             | 0
9       | Realloc cap=16   | 8 (copy 8 old elements)
...

Total copies for n pushes ≈ n + n/2 + n/4 + ... ≈ 2n
Average per push = 2n/n = O(1)
```

---

## Avoiding Reallocations

If you know the size upfront, pre-allocate:

```rust
fn main() {
    // BAD: starts at 0, reallocates multiple times
    let mut v = Vec::new();
    for i in 0..10_000 {
        v.push(i);
    }
    
    // GOOD: single allocation
    let mut v = Vec::with_capacity(10_000);
    for i in 0..10_000 {
        v.push(i);
    }
    
    // BEST: use collect (also pre-allocates when iterator has size_hint)
    let v: Vec<i32> = (0..10_000).collect();
    
    // Reserve more space for existing vec
    let mut v = vec![1, 2, 3];
    v.reserve(100);  // Ensure room for 100 MORE elements
    println!("cap >= {}", v.capacity());  // At least 103
}
```

---

## Shrinking

After removing many elements, capacity stays large:

```rust
fn main() {
    let mut v: Vec<i32> = (0..1000).collect();
    println!("len: {}, cap: {}", v.len(), v.capacity());
    // len: 1000, cap: 1000
    
    v.truncate(10);
    println!("len: {}, cap: {}", v.len(), v.capacity());
    // len: 10, cap: 1000  (still huge!)
    
    v.shrink_to_fit();
    println!("len: {}, cap: {}", v.len(), v.capacity());
    // len: 10, cap: 10  (freed excess memory)
    
    // shrink_to: keep at least this much
    let mut v: Vec<i32> = Vec::with_capacity(100);
    v.extend(0..10);
    v.shrink_to(20);  // cap = at least 20 (not less than len)
}
```

---

## Zero-Sized Types

Types with size 0 (like `()`) are special:

```rust
fn main() {
    let v: Vec<()> = vec![(); 1_000_000];
    println!("len: {}", v.len());            // 1000000
    println!("cap: {}", v.capacity());       // usize::MAX
    println!("size: {}", std::mem::size_of_val(&*v));  // 0
    // No heap allocation at all!
}
```

---

## Memory Layout Diagram

```
Vec<String> with 3 elements:

Stack (Vec struct):
┌──────────┐
│ ptr ──────────┐
│ len: 3   │    │
│ cap: 4   │    │
└──────────┘    │
                ▼
Heap (Vec buffer, 4 × size_of::<String>() = 4 × 24 = 96 bytes):
┌──────────────┬──────────────┬──────────────┬──────────────┐
│  String #0   │  String #1   │  String #2   │  (unused)    │
│ ptr,len,cap  │ ptr,len,cap  │ ptr,len,cap  │              │
│  ↓           │  ↓           │  ↓           │              │
└──┼───────────┴──┼───────────┴──┼───────────┴──────────────┘
   ▼              ▼              ▼
 "hello"       "world"        "rust"   ← Each String has its OWN heap alloc

Total heap allocations: 4 (1 for Vec buffer + 3 for Strings)
```

---

## Exercises

### Exercise 1: Track Capacity

Write code that pushes 20 elements and prints every time the capacity changes:

<details>
<summary>Solution</summary>

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    let mut prev_cap = v.capacity();
    
    for i in 0..20 {
        v.push(i);
        if v.capacity() != prev_cap {
            println!(
                "After push #{}: cap changed from {} to {}",
                i + 1,
                prev_cap,
                v.capacity()
            );
            prev_cap = v.capacity();
        }
    }
    println!("Final: len={}, cap={}", v.len(), v.capacity());
}
```

</details>

### Exercise 2: Pre-allocate

Convert this function to avoid any reallocations:

```rust
fn squares(n: usize) -> Vec<i64> {
    let mut result = Vec::new();
    for i in 0..n {
        result.push((i as i64) * (i as i64));
    }
    result
}
```

<details>
<summary>Solution</summary>

```rust
fn squares(n: usize) -> Vec<i64> {
    // Option 1: with_capacity
    let mut result = Vec::with_capacity(n);
    for i in 0..n {
        result.push((i as i64) * (i as i64));
    }
    result
    
    // Option 2: collect (also pre-allocates via size_hint)
    // (0..n).map(|i| (i as i64) * (i as i64)).collect()
}
```

</details>

---

## Summary

| Concept | Detail |
|---------|--------|
| Stack size | 24 bytes (ptr + len + cap) on 64-bit |
| Heap buffer | Contiguous, `capacity × size_of::<T>()` bytes |
| Growth | Approximately doubles capacity |
| Amortized push | O(1) |
| Reallocation | Copies all elements, invalidates references |
| Pre-allocate | `Vec::with_capacity(n)` avoids repeated growth |
| Shrink | `shrink_to_fit()` releases unused capacity |

---

**Previous:** [← Vectors](./01-vectors.md) · **Next:** [String Internals →](./03-string-internals.md)

<p align="center"><i>Tutorial 2 of 8 — Stage 5: Collections</i></p>

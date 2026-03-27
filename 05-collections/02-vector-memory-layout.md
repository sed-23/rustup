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

### The Allocator — What Happens When Vec Needs More Memory

When `Vec::push` triggers a resize, what actually happens at the system level? Understanding the allocator turns "it just works" into a mental model you can reason about.

**The allocation pipeline:**

```
Vec::push()
  │
  ├─ len < capacity? → write element, increment len (≈5ns) ✅
  │
  └─ len == capacity? → must grow:
       │
       ├─ 1. Calculate new capacity (double / next power of 2)
       ├─ 2. Call global allocator: alloc::alloc::alloc(new_layout)
       │     │
       │     ├─ Small alloc (≤4KB): thread-local cache → instant (~100ns)
       │     ├─ Medium alloc: central free-list → fast (~200-500ns)
       │     └─ Large alloc: OS syscall (mmap/VirtualAlloc) → slow (~1-10μs)
       │
       ├─ 3. memcpy old buffer → new buffer (O(n))
       ├─ 4. Deallocate old buffer
       └─ 5. Write new element, update ptr/len/cap
```

**Size classes and the free list:**

Modern allocators (glibc `malloc`, jemalloc, mimalloc) don't manage memory as one big block. They organize free memory into **size classes** — bins of pre-divided chunks:

| Size Class | Typical Bin Sizes | Source |
|---|---|---|
| Tiny | 8, 16, 32, 64 bytes | Thread-local cache |
| Small | 128, 256, 512, 1024 bytes | Thread-local cache |
| Medium | 2KB, 4KB, 8KB | Central arena |
| Large | 16KB, 32KB, ... | mmap / VirtualAlloc |

When `Vec` needs 64 bytes, the allocator grabs a pre-carved 64-byte slot from the thread-local cache — no locks, no syscalls, near-instant.

**Why `Vec::with_capacity(n)` matters — real numbers:**

```rust
// Without pre-allocation: building a Vec of 1 million elements
// Reallocations: ~20 (log₂ 1,000,000 ≈ 20)
// Total bytes copied: ≈ 2,000,000 × size_of::<T>()
// Total alloc calls: ~20
// Estimated overhead: ~40μs in allocator + memcpy time

// With pre-allocation:
// Reallocations: 0
// Total alloc calls: 1
// Estimated overhead: ~0.5μs (one allocation)
```

That's roughly an **80× reduction** in allocation overhead for this case.

**Rust's allocator story:**

- **Before Rust 1.32:** jemalloc was the default allocator (great for multi-threaded workloads)
- **Rust 1.32+:** System allocator is the default (`malloc` on Linux, `HeapAlloc` on Windows)
- **You can switch:** Use `#[global_allocator]` to plug in jemalloc, mimalloc, or your own custom allocator

```rust
// Example: using jemalloc (add jemallocator to Cargo.toml)
// use jemallocator::Jemalloc;
// #[global_allocator]
// static GLOBAL: Jemalloc = Jemalloc;
```

> **Takeaway:** A `push` that fits in existing capacity is ~5ns (a pointer write). A `push` that triggers reallocation is ~100-500ns+ (allocator call + memcpy). Pre-allocating when you know the size eliminates the expensive path entirely.

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

### The History of Dynamic Array Growth Strategies

The doubling strategy isn't accidental — it's the result of decades of computer science research and practical engineering tradeoffs.

**Why doubling (×2) dominates:**

Most major implementations use a factor of 2: Rust's `Vec`, Java's `ArrayList`, C++'s `std::vector` (GCC/Clang), and Python's `list`. The reason is mathematical: with a ×2 growth factor, each element is copied at most $O(\log n)$ times across all reallocations, giving a total copy cost of $\sum_{i=0}^{\log_2 n} \frac{n}{2^i} \approx 2n$, which means **amortized O(1) per append**.

**The landscape of growth factors:**

| Implementation | Growth Factor | Rationale |
|---|---|---|
| Rust `Vec` | ×2 (next power-of-2) | Simple, fast, allocator-friendly |
| Java `ArrayList` | ×1.5 (`old + old >> 1`) | Reduces peak memory waste |
| C# `List<T>` | ×2 exactly | Simplicity, predictability |
| Python `list` | ~×1.125 (over-allocates by ~12.5%) | Small objects, gradual growth |
| Firefox SpiderMonkey | ×1.5 | Reduces wasted space vs ×2 |
| Facebook `folly::fbvector` | ×1.5 | Allows in-place realloc more often |

**Why does Facebook use 1.5×?** When you grow by exactly 2×, the new allocation is always *larger* than the sum of all previous freed blocks — so the allocator can never reuse that freed space for the new buffer. With 1.5× growth, after a few reallocations the freed blocks *add up* to enough space for the next allocation, enabling in-place extension.

**The golden ratio connection:**

Growth factor $\phi \approx 1.618$ (the golden ratio) is theoretically optimal for memory reuse. After two frees, the combined freed space exactly equals the next allocation size. Some custom allocators use this, but in practice the difference from 1.5× is negligible.

**Rust's actual strategy in detail:**

Rust's `Vec` doesn't strictly double — it requests the *next power of 2* capacity, or at minimum doubles. For the first allocation, it picks a small starting capacity (4 for types ≤ 1KB). This power-of-2 alignment is friendly to memory allocators, which typically organize free blocks in size classes that are powers of 2.

```
Growth factor tradeoff:

  Larger factor (e.g., 2×)          Smaller factor (e.g., 1.25×)
  ├─ Fewer reallocations             ├─ More reallocations
  ├─ More wasted memory              ├─ Less wasted memory
  └─ Simpler math                    └─ Better allocator reuse

  Sweet spot for most apps: 1.5× – 2×
  For embedded/real-time: carefully tuned per workload
```

> **Rule of thumb:** For 99% of applications, Rust's default doubling strategy is ideal. Only tune allocation strategy in embedded systems, real-time audio/video, database engines, or when profiling reveals allocation as a bottleneck.

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

### Zero-Sized Types — A Rust Superpower

ZSTs are one of Rust's most elegant emergent properties — types that exist at compile time but occupy **zero bytes** at runtime.

**What qualifies as a ZST?**

```rust
use std::mem::size_of;

struct Marker;              // size = 0
struct Empty {}             // size = 0
enum Void {}                // size = 0 (uninhabited)

assert_eq!(size_of::<()>(), 0);
assert_eq!(size_of::<Marker>(), 0);
assert_eq!(size_of::<std::marker::PhantomData<String>>(), 0);
```

**How Vec handles ZSTs — no special cases needed:**

```
Vec<i32> with 3 elements:           Vec<()> with 1,000,000 elements:
┌──────────────┐                   ┌──────────────────────────┐
│ ptr → [heap] │                   │ ptr → NonNull::dangling() │  ← special
│ len: 3       │                   │ len: 1_000_000            │     sentinel,
│ cap: 4       │                   │ cap: usize::MAX           │     NOT a real
└──────────────┘                   └──────────────────────────┘     heap pointer
  Heap: 16 bytes allocated           Heap: 0 bytes allocated!
```

Rust uses `NonNull::dangling()` — a well-aligned, non-null pointer that is never dereferenced — as the buffer pointer for ZST vectors. Since each element is 0 bytes, the capacity is effectively infinite (`usize::MAX`).

**Why this matters in the real world:**

The standard library exploits ZSTs extensively:

| Type | Implementation | ZST Usage |
|---|---|---|
| `HashSet<T>` | `HashMap<T, ()>` | Values are `()` → zero space for values |
| `BTreeSet<T>` | `BTreeMap<T, ()>` | Same trick |
| `LinkedList` internal | Node markers | Sentinel nodes with no data |

```rust
// HashSet<String> is secretly HashMap<String, ()>
// The () values take ZERO additional space per entry!
use std::collections::HashSet;
let mut set = HashSet::new();
set.insert("hello".to_string());  // Only stores the key, no value overhead
```

**The typestate pattern — compile-time state machines at zero cost:**

```rust
use std::marker::PhantomData;

struct Locked;
struct Unlocked;

struct Door<State> {
    _state: PhantomData<State>,  // 0 bytes!
}

impl Door<Locked> {
    fn unlock(self) -> Door<Unlocked> { Door { _state: PhantomData } }
}
impl Door<Unlocked> {
    fn lock(self) -> Door<Locked> { Door { _state: PhantomData } }
    fn open(&self) { println!("Door opened!"); }
}
// door.open() only compiles if door is Door<Unlocked> — checked at compile time!
```

**Comparison with other languages:**

| Language | Minimum Object Size | Notes |
|---|---|---|
| **Rust** | **0 bytes** (ZSTs) | Monomorphization specializes code per type |
| C++ | 1 byte minimum | Empty base optimization can help |
| Java | 16 bytes | Object header (mark word + class pointer) |
| Python | ~56 bytes | `object()` overhead |
| Go | 0 bytes for `struct{}` | Similar to Rust, but no generics specialization |

Rust's monomorphization means `Vec<()>` gets its own specialized machine code that **skips allocation entirely** — the compiler doesn't just store zero bytes, it removes the allocation logic from the generated code.

> **Key insight:** ZSTs aren't a hack or special case — they fall out naturally from Rust's type system and monomorphization. When `size_of::<T>() == 0`, all the math (capacity × 0 = 0 bytes needed) just works.

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

# The Iterator Trait 🔄

> **Iterators are Rust's way of processing sequences of values lazily. The `Iterator` trait has one required method — `next()` — and provides dozens of powerful built-in methods for transforming, filtering, and consuming data.**

---

## Table of Contents

- [What Is an Iterator?](#what-is-an-iterator)
- [The Iterator Trait](#the-iterator-trait)
- [Creating Iterators](#creating-iterators)
- [iter, iter_mut, into_iter](#iter-iter_mut-into_iter)
- [Laziness](#laziness)
- [for Loops and Iterators](#for-loops-and-iterators)
- [IntoIterator Trait](#intoiterator-trait)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is an Iterator?

An iterator is anything that produces a sequence of values, one at a time:

```rust
fn main() {
    let v = vec![10, 20, 30];
    
    // Create an iterator
    let mut iter = v.iter();
    
    // Call next() to get each value
    println!("{:?}", iter.next());  // Some(&10)
    println!("{:?}", iter.next());  // Some(&20)
    println!("{:?}", iter.next());  // Some(&30)
    println!("{:?}", iter.next());  // None (exhausted)
    println!("{:?}", iter.next());  // None (stays None forever)
}
```

---

### The History of Iterators — From CLU to Rust

Iterators didn't spring from nowhere. Their evolution spans five decades of programming language design — and Rust's `Iterator` trait is the culmination of lessons learned from every prior attempt.

#### The Timeline

**CLU (1975)** — Barbara Liskov and her team at MIT created CLU, the **first language with generator-like iterators**. CLU introduced the `yield` statement, allowing functions to produce a sequence of values one at a time. This was revolutionary — before CLU, producing a sequence meant building the entire thing in memory first.

```
// CLU iterator (1975 syntax)
gen_primes = iter() yields (int)
    // ...yield successive primes...
end gen_primes
```

**Smalltalk (1980)** — Alan Kay's Smalltalk took a different path: **collection methods**. Instead of external iterators, Smalltalk gave collections internal iteration methods like `do:`, `select:`, `collect:`, and `reject:`. These are the direct ancestors of Rust's `.for_each()`, `.filter()`, `.map()`, and `.filter(|x| !predicate(x))`. Smalltalk proved that declarative iteration could be elegant.

**C++ STL (1994, Alexander Stepanov)** — The Standard Template Library introduced **five categories** of iterators: Input, Output, Forward, Bidirectional, and RandomAccess. This was the first attempt at generic, zero-cost iteration in a systems language. The design was powerful but complex — you needed to understand the category hierarchy, and misusing an iterator category led to subtle bugs or silent performance degradation.

**Java Iterator (1998)** — Java 1.2 introduced `Iterator<T>` with `hasNext()` and `next()`, plus `Iterable<T>` so collections could be used in enhanced for-loops. Simple and clean, but the two-method design (`hasNext` + `next`) creates a protocol that callers can violate at runtime.

**Python Generators (2001, PEP 255)** — Guido van Rossum brought CLU's `yield` to the mainstream. Python generators let any function become a lazy iterator by using `yield` instead of `return`. This was **directly inspired by CLU** — PEP 255 explicitly cites Liskov's work. Generators made lazy iteration accessible to millions of developers.

**Rust's Iterator Trait (2015)** — Rust synthesized the best ideas: CLU's laziness, Smalltalk's rich method vocabulary, C++'s zero-cost ambition, and Java's simplicity. The result? **ONE trait, ONE required method (`next()`), ~75 provided methods**, and zero runtime cost.

#### Comparison: Iterator Design Across Languages

| Language | Year | Iterator Design | Lazy? | # of Iterator Types |
|------------|------|------------------------------------------|-------|---------------------|
| CLU | 1975 | `yield`-based generators | Yes | 1 (generators) |
| Smalltalk | 1980 | Collection methods (`do:`, `select:`) | No | Internal only |
| C++ STL | 1994 | 5 categories (Input→RandomAccess) | No | 5 |
| Java | 1998 | `Iterator<T>` (hasNext/next) | No | 1 + Iterable |
| Python | 2001 | `__iter__`/`__next__` + generators | Generators only | 1 + generators |
| JavaScript | 2015 | `Symbol.iterator` + `function*` | Generators only | 1 + generators |
| **Rust** | **2015** | **`Iterator` trait (`next()` → `Option`)** | **Yes** | **1 (with ~75 methods)** |

#### Why Rust's Design Wins

C++ needed five iterator categories because its iterators are modeled after **pointers** — different pointer capabilities (read-only, write-only, forward, backward, random jump) required different types. Rust's `Iterator` trait doesn't carry this baggage. Instead of encoding capabilities in the type hierarchy, Rust provides **optional methods** with default implementations. The `size_hint()` method lets iterators *optionally* report their length — no separate "RandomAccess" category needed. And the type system guarantees correctness without the five-tier complexity.

The result: Rust's iterator system is **simpler to learn** than C++ STL iterators, **safer** than Java/Python iterators, and **just as fast** as hand-written C.

---

### Iterators Across Programming Languages

The iterator pattern is one of the 23 original **Gang of Four design patterns** (1994). Nearly every modern language implements it — but each makes different trade-offs.

#### How Languages Handle Iteration

```
  Java                Python              C++
  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐
  │ Iterator<T>  │    │ __iter__()   │    │ InputIterator            │
  │  hasNext()   │    │ __next__()   │    │ OutputIterator           │
  │  next()      │    │ StopIteration│    │ ForwardIterator          │
  └──────────────┘    └──────────────┘    │ BidirectionalIterator    │
                                          │ RandomAccessIterator     │
  JavaScript          Go                  └──────────────────────────┘
  ┌──────────────┐    ┌──────────────┐
  │ Symbol.iter  │    │ No protocol! │    Haskell
  │ function*    │    │ for range on │    ┌──────────────┐
  │ { value,     │    │ maps/slices/ │    │ All lists    │
  │   done }     │    │ strings only │    │ are lazy     │
  └──────────────┘    └──────────────┘    └──────────────┘

  Rust
  ┌─────────────────────────────────────┐
  │ Iterator trait — ONE required method│
  │ fn next(&mut self) -> Option<Item>  │
  │ None means done. No exceptions.     │
  │ No boolean checks. Just a value.    │
  └─────────────────────────────────────┘
```

- **Java**: `Iterator<T>` interface requires two methods — `hasNext()` and `next()`. You must check `hasNext()` before calling `next()`, and a runtime `NoSuchElementException` is thrown if you forget.
- **Python**: `__iter__()` returns the iterator, `__next__()` produces values. Termination is signaled by raising a `StopIteration` exception — an exception for normal control flow.
- **C++**: Five iterator categories form a complex hierarchy. Raw pointer arithmetic is valid iteration. It's powerful but error-prone.
- **JavaScript**: `Symbol.iterator` returns an object with `next()` that produces `{ value, done }`. Generator functions (`function*` with `yield`) simplify creation.
- **Go**: No iterator protocol at all. `for range` works on maps, slices, channels, and strings — for everything else, you write explicit loops or use channels.
- **Haskell**: All lists are lazy by default — every list is effectively an iterator. The list monad provides chaining.
- **Rust**: One required method (`next()`), returns `Option<Item>`. `None` signals the end — no exception, no boolean check, just a value in the type system.

#### The `Option` Approach: Why Rust's Termination Signal Is Superior

Rust doesn't throw an exception (Python) or require a separate check (Java). The end-of-iteration signal is **encoded in the return type**. You literally cannot forget to handle it — the compiler enforces it through `Option<T>`.

#### Comparison Table

| Language | Iterator Interface | Termination Signal | Lazy by Default? | Zero-Cost? |
|------------|------------------------------|------------------------------|------------------|------------|
| Java | `Iterator<T>` (hasNext/next) | `NoSuchElementException` | No | No (boxing) |
| Python | `__iter__` / `__next__` | `StopIteration` exception | No (generators yes) | No |
| C++ | 5 iterator categories | Comparison with `end()` | No | Yes |
| JavaScript | `Symbol.iterator` / `next()`| `{ done: true }` | No (generators yes) | No |
| Go | None (channels/for range) | Channel close / range end | No | N/A |
| Haskell | List monad / typeclass | Empty list `[]` | Yes | No (GC) |
| **Rust** | **`Iterator` trait (`next`)** | **`None`** | **Yes** | **Yes** |

Rust uniquely combines **laziness by default**, **zero-cost abstractions** (no heap allocation, no vtable dispatch after monomorphization), and **ownership integration** — iterators respect borrowing rules at compile time.

---

## The Iterator Trait

```rust
pub trait Iterator {
    type Item;  // The type of each element
    
    fn next(&mut self) -> Option<Self::Item>;
    
    // ... 70+ provided methods (map, filter, fold, etc.)
}
```

Only `next()` is required. Everything else is built on top of it.

The `Item` associated type tells you what the iterator yields:

| Iterator | `Item` type |
|----------|-------------|
| `vec.iter()` | `&T` |
| `vec.iter_mut()` | `&mut T` |
| `vec.into_iter()` | `T` |
| `"hello".chars()` | `char` |
| `"hello".bytes()` | `u8` |
| `(0..10)` | `i32` |
| `hashmap.iter()` | `(&K, &V)` |

---

## Creating Iterators

```rust
fn main() {
    // From ranges
    let r = 0..5;           // 0, 1, 2, 3, 4
    let r = 0..=5;          // 0, 1, 2, 3, 4, 5
    let r = (0..10).step_by(2);  // 0, 2, 4, 6, 8
    
    // From collections
    let v = vec![1, 2, 3];
    let iter = v.iter();
    
    // From functions
    let ones = std::iter::repeat(1);           // 1, 1, 1, 1, ...
    let empty: std::iter::Empty<i32> = std::iter::empty();  // nothing
    let single = std::iter::once(42);          // just 42
    
    // From a closure (generate values)
    let mut count = 0;
    let counter = std::iter::from_fn(move || {
        count += 1;
        if count <= 5 { Some(count) } else { None }
    });
    let v: Vec<i32> = counter.collect();
    println!("{:?}", v);  // [1, 2, 3, 4, 5]
    
    // Successors (each value computed from previous)
    let powers: Vec<i32> = std::iter::successors(Some(1), |&prev| {
        let next = prev * 2;
        if next <= 128 { Some(next) } else { None }
    }).collect();
    println!("{:?}", powers);  // [1, 2, 4, 8, 16, 32, 64, 128]
}
```

---

## iter, iter_mut, into_iter

Every Rust collection provides three iterator methods:

### `.iter()` — borrows immutably

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    for val in v.iter() {
        // val is &i32
        println!("{}", val);
    }
    
    println!("{:?}", v);  // ✅ v still usable
}
```

### `.iter_mut()` — borrows mutably

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    
    for val in v.iter_mut() {
        // val is &mut i32
        *val *= 10;
    }
    
    println!("{:?}", v);  // [10, 20, 30]
}
```

### `.into_iter()` — takes ownership

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];
    
    for val in v.into_iter() {
        // val is String (owned)
        println!("{}", val);
    }
    
    // println!("{:?}", v);  // ❌ v was consumed
}
```

### Comparison Table

```
Method        | Item type | Ownership
--------------+-----------+------------------
.iter()       | &T        | Borrows collection
.iter_mut()   | &mut T    | Mutably borrows
.into_iter()  | T         | Consumes collection
```

---

### iter(), iter_mut(), into_iter() — The Three Iterators Deep Dive

This is one of the most confusing topics for Rust beginners. Why does Rust have **three** iteration methods when every other language has one?

The answer is **ownership**. Each method corresponds to one of Rust's three ways of accessing data:

```
                    ┌───────────────────────────────────────────────┐
                    │           The Collection (Vec<T>)             │
                    │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐        │
                    │  │  A  │  │  B  │  │  C  │  │  D  │        │
                    │  └─────┘  └─────┘  └─────┘  └─────┘        │
                    └───────────────────────────────────────────────┘
                          │           │           │
         .iter()          │  .iter_mut()          │  .into_iter()
         Borrows &T       │  Borrows &mut T       │  Takes ownership T
              ↓           │       ↓               │       ↓
         You can READ     │  You can MODIFY       │  You OWN each item
         Collection lives │  Collection lives     │  Collection is GONE
```

#### The `for` Loop Sugar

The `for` loop chooses which iterator based on how you write it:

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    for x in &v     { /* .iter()      — x: &i32     — v still alive */ }
    for x in &mut v { /* .iter_mut()  — x: &mut i32 — v still alive */ }
    for x in v      { /* .into_iter() — x: i32      — v is CONSUMED */ }
    // v is gone after the last loop!
}
```

Why does `for x in v` **consume** the collection by default? Because most loops process data and discard the source — this is the common case. Rust optimizes for the common path and makes you be explicit (`&v`) when you want to keep the collection.

#### No Other Language Has This Distinction

No other mainstream language distinguishes between these three modes:

- **Java / Python**: Iterating a collection borrows it implicitly. The garbage collector keeps the collection alive regardless. You can never "consume" a collection by iterating.
- **C++**: Range-based `for (const auto& x : vec)` is closest to Rust's `for x in &vec`. But there's no ownership transfer — destructors run at scope end, not at iteration.
- **Rust**: The three-iterator pattern reflects ownership semantics. It repeats across **every** collection type: `Vec`, `HashMap`, `BTreeMap`, `HashSet`, `LinkedList`, `VecDeque`, and any custom types that implement `IntoIterator`.

#### Quick Reference

| Syntax | Calls | Yields | Collection After? |
|------------------|---------------|-----------|-------------------|
| `for x in &col` | `.iter()` | `&T` | Still usable |
| `for x in &mut col` | `.iter_mut()` | `&mut T` | Still usable (modified) |
| `for x in col` | `.into_iter()` | `T` | **Consumed / gone** |

When in doubt: use `&collection` in your `for` loop. Only use `collection` directly when you intentionally want to give up ownership.

---

## Laziness

**Iterator adaptors are lazy** — they do nothing until consumed:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // This does NOTHING yet — just builds a pipeline
    let pipeline = v.iter()
        .map(|x| {
            println!("mapping {}", x);  // This won't print!
            x * 2
        })
        .filter(|x| {
            println!("filtering {}", x);  // This won't print either!
            x > &4
        });
    
    println!("Pipeline created, nothing computed yet!");
    
    // NOW it runs (collect is a consumer)
    let result: Vec<&i32> = pipeline.collect();
    // mapping 1
    // filtering 2
    // mapping 2
    // filtering 4
    // mapping 3
    // filtering 6   ← kept
    // mapping 4
    // filtering 8   ← kept
    // mapping 5
    // filtering 10  ← kept
    
    // Note: items are processed ONE AT A TIME through the whole chain
    // NOT: map all, then filter all
}
```

Benefits of laziness:
- No intermediate collections allocated
- Can short-circuit (e.g., `find`, `any`, `take`)
- Can work with infinite iterators

```rust
fn main() {
    // Infinite iterator — only takes what's needed
    let first_10_squares: Vec<i32> = (0..)
        .map(|x| x * x)
        .take(10)
        .collect();
    
    println!("{:?}", first_10_squares);
    // [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
}
```

---

### Lazy Evaluation — Why Iterators Don't Do Work Until Asked

When you write `vec.iter().map(|x| x * 2).filter(|x| x > &5)`, **nothing happens**. No multiplication. No filtering. Zero computation.

The iterator chain just builds a **nested type** — a data structure describing the work to be done:

```
  vec.iter().map(|x| x * 2).filter(|x| x > &5)

  Compiles to a type like:
  ┌─────────────────────────────────────────────────────┐
  │ Filter<                                             │
  │   Map<                                              │
  │     Iter<Vec<i32>>,     ← source                    │
  │     |x| x * 2           ← transform (stored)        │
  │   >,                                                │
  │   |x| x > &5            ← predicate (stored)        │
  │ >                                                   │
  └─────────────────────────────────────────────────────┘

  No work has been done. This is just a struct.
```

Work only happens when a **consumer** is called: `.collect()`, `.sum()`, `.for_each()`, `.count()`, `.find()`, `.any()`, `.fold()`.

#### Why Laziness Matters for Performance

**1. Early Termination**
```rust
fn main() {
    let v = vec![1, 50, 3, 200, 5, 8];
    // Stops as soon as it finds 200 — never looks at 5 or 8
    let big = v.iter().find(|&&x| x > 100);
    println!("{:?}", big);  // Some(200)
}
```

**2. Loop Fusion** — the compiler merges `map` + `filter` into a single loop. No intermediate `Vec` is allocated. Each element flows through the **entire chain** before the next element starts:

```
  Element-at-a-time processing (lazy):

  Source: [1, 2, 3, 4, 5]
          │
          ├─ 1 → map → 2  → filter (2 > 5? No)  → discarded
          ├─ 2 → map → 4  → filter (4 > 5? No)  → discarded
          ├─ 3 → map → 6  → filter (6 > 5? Yes) → collected ✓
          ├─ 4 → map → 8  → filter (8 > 5? Yes) → collected ✓
          └─ 5 → map → 10 → filter (10 > 5? Yes)→ collected ✓

  vs. Eager evaluation (Python list comprehensions):

  Step 1: map ALL  → [2, 4, 6, 8, 10]    ← allocates a new list
  Step 2: filter ALL → [6, 8, 10]          ← allocates ANOTHER list
```

**3. Pipeline Efficiency** — no intermediate allocations means less memory pressure and better cache locality.

#### Rust vs. Other Languages

- **Python list comprehensions** (`[x*2 for x in arr]`) are **eager** — they create a full list in memory. Python generators (`yield`) are lazy but carry runtime overhead for the generator state machine.
- **Java Streams** (`.stream().map().filter()`) are lazy, but have boxing overhead for primitives and vtable dispatch for lambda captures.
- **Rust iterators** are **zero-cost**: after monomorphization and optimization, the code is **identical** to a hand-written `for` loop.

#### The Zero-Cost Promise

This is the key Rust guarantee: *"Abstractions that compile down to the same code you'd write by hand."*

```rust
// These two produce the EXACT same assembly:

// Version 1: Iterator chain
fn sum_doubled_evens_iter(v: &[i32]) -> i32 {
    v.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * 2)
        .sum()
}

// Version 2: Hand-written loop
fn sum_doubled_evens_loop(v: &[i32]) -> i32 {
    let mut acc = 0;
    for &x in v {
        if x % 2 == 0 {
            acc += x * 2;
        }
    }
    acc
}
```

The compiler sees through the iterator abstractions completely. There's no function call overhead, no heap allocation, no indirection — just a tight loop with an accumulator and a branch.

---

### Zero-Cost Iterators — How Rust Matches Hand-Written Loops

The phrase "zero-cost abstraction" gets thrown around a lot. Let's see exactly what it means for iterators — and **why** it works.

#### The Claim

This iterator chain:

```rust
fn sum_filtered(v: &[i32]) -> i32 {
    v.iter()
        .map(|x| x * 2)
        .filter(|x| x > &5)
        .sum::<i32>()
}
```

Compiles to **the exact same assembly** as this hand-written loop:

```rust
fn sum_filtered_manual(v: &[i32]) -> i32 {
    let mut total = 0;
    for &x in v {
        let doubled = x * 2;
        if doubled > 5 {
            total += doubled;
        }
    }
    total
}
```

You can verify this yourself on [godbolt.org](https://godbolt.org) — both functions produce identical x86-64 assembly under `-O2` or `--release`.

#### How? Monomorphization

When you write `v.iter().map(|x| x * 2).filter(|x| x > &5)`, the compiler creates a **unique concrete type** for this specific chain:

```
Filter<Map<std::slice::Iter<'_, i32>, [closure@map]>, [closure@filter]>
```

Each closure is a unique, zero-sized type. The compiler **generates specialized code** for this exact combination — there's no vtable, no dynamic dispatch, no trait object. The `next()` calls are all **inlined**, and LLVM then optimizes the nested function calls into a single flat loop.

#### What the Compiler "Sees" After Optimization

Conceptually, the compiler unrolls the iterator chain into:

```rust
// What LLVM produces (pseudocode):
fn sum_filtered_optimized(v: &[i32]) -> i32 {
    let mut acc = 0i32;
    let ptr = v.as_ptr();
    let end = ptr.add(v.len());
    while ptr < end {
        let val = *ptr * 2;  // map inlined
        if val > 5 {          // filter inlined
            acc += val;        // sum inlined
        }
        ptr = ptr.add(1);
    }
    acc
}
```

Three layers of abstraction (`Map`, `Filter`, `Sum`) collapse into **a single loop with a branch and an add**.

#### LLVM Auto-Vectorization: Even Faster Than You'd Write

Here's the surprising part: iterator chains can sometimes be **faster** than naïve hand-written loops. LLVM's auto-vectorizer can convert iterator chains into **SIMD instructions** — processing 4, 8, or even 16 elements simultaneously:

```
// Scalar (one element at a time):
add eax, [rsi]        // ~1 element per cycle

// SIMD vectorized (8 elements at a time with AVX2):
vpaddd ymm0, ymm0, [rsi]   // ~8 elements per cycle!
```

Iterator chains give the compiler **more information** about data flow than manual loops with mutable variables, making auto-vectorization easier to apply.

#### Why This Doesn't Work in Other Languages

| Language | Why iterators have overhead |
|----------|----------------------------|
| **Java** | `Iterator<T>` uses virtual dispatch (`next()` is a vtable call). Primitives are boxed into `Integer`, `Double`, etc. Stream pipeline objects are heap-allocated. |
| **Python** | Every iterator call goes through the interpreter. `__next__()` is a dynamic method lookup. Generator state is stored on the heap. Each value is a heap-allocated object. |
| **JavaScript** | Iterator protocol uses dynamic dispatch. `{ value, done }` objects are allocated per element. JIT can optimize hot loops but not guaranteed. |
| **Rust** | Monomorphization eliminates all abstraction. Closures are zero-sized types. No heap allocation. No virtual dispatch. Identical to C. |

#### Benchmark Reality

In real-world benchmarks, Rust iterator chains typically perform **within 0-2%** of equivalent C `for` loops, and sometimes **outperform** them when LLVM's auto-vectorizer kicks in. This is what "zero-cost abstractions" means in practice: you pay **nothing** for using the high-level API. Write the readable version — the compiler handles the rest.

---

## for Loops and Iterators

`for` loops automatically call `.into_iter()`:

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    // These are equivalent:
    for x in v.iter() { /* x: &i32 */ }
    for x in &v { /* x: &i32 — &v calls v.iter() */ }
    
    let mut v = vec![1, 2, 3];
    for x in v.iter_mut() { /* x: &mut i32 */ }
    for x in &mut v { /* x: &mut i32 */ }
    
    let v = vec![1, 2, 3];
    for x in v.into_iter() { /* x: i32, v consumed */ }
    // for x in v { /* same thing */ }
}
```

Under the hood, a `for` loop desugars to:

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    // This:
    for x in &v {
        println!("{}", x);
    }
    
    // Is equivalent to:
    let mut iter = (&v).into_iter();
    loop {
        match iter.next() {
            Some(x) => println!("{}", x),
            None => break,
        }
    }
}
```

---

## IntoIterator Trait

`IntoIterator` converts something INTO an iterator:

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;
    
    fn into_iter(self) -> Self::IntoIter;
}
```

This is what `for` loops use. It's implemented for:
- `Vec<T>` → yields owned `T`
- `&Vec<T>` → yields `&T`
- `&mut Vec<T>` → yields `&mut T`
- Arrays, slices, HashMap, HashSet, BTreeMap, etc.

You can accept any iterable in your functions:

```rust
fn sum_all<I: IntoIterator<Item = i32>>(values: I) -> i32 {
    values.into_iter().sum()
}

fn main() {
    println!("{}", sum_all(vec![1, 2, 3]));     // 6
    println!("{}", sum_all([1, 2, 3]));          // 6
    println!("{}", sum_all(1..=3));              // 6
}
```

---

### IntoIterator and the Orphan Rule — Why `for` Loops Just Work

Ever wondered why all of these compile without any extra work?

```rust
fn main() {
    let v = vec![1, 2, 3];

    for x in &v {}       // iterates over &i32
    for x in &mut v.clone() {}  // iterates over &mut i32
    for x in v {}        // iterates over i32 (consumes v)
}
```

The magic is `IntoIterator` — and Rust's standard library implements it **three times** for every collection.

#### The Desugaring

When the compiler sees `for x in something`, it rewrites it to:

```rust
let mut iter = IntoIterator::into_iter(something);
loop {
    match iter.next() {
        Some(x) => { /* loop body */ },
        None => break,
    }
}
```

So `for x in &v` calls `IntoIterator::into_iter(&v)`, and `for x in v` calls `IntoIterator::into_iter(v)`. The **type** of the expression after `in` determines which `IntoIterator` impl is selected.

#### The Three Implementations for `Vec<T>`

| Expression | Type passed to `into_iter` | `IntoIterator` impl | Returns | Yields |
|------------|---------------------------|---------------------|---------|--------|
| `for x in &vec` | `&Vec<T>` | `impl IntoIterator for &Vec<T>` | `std::slice::Iter<T>` | `&T` |
| `for x in &mut vec` | `&mut Vec<T>` | `impl IntoIterator for &mut Vec<T>` | `std::slice::IterMut<T>` | `&mut T` |
| `for x in vec` | `Vec<T>` | `impl IntoIterator for Vec<T>` | `std::vec::IntoIter<T>` | `T` |

This pattern is **repeated for every standard library collection**: `HashMap`, `BTreeMap`, `HashSet`, `BTreeSet`, `LinkedList`, `VecDeque`, and more. Each one provides all three implementations.

#### Making Your Own Types Work with `for` Loops

You can implement `IntoIterator` for your own types:

```rust
struct Countdown(u32);

struct CountdownIter(u32);

impl Iterator for CountdownIter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.0 == 0 {
            None
        } else {
            let val = self.0;
            self.0 -= 1;
            Some(val)
        }
    }
}

impl IntoIterator for Countdown {
    type Item = u32;
    type IntoIter = CountdownIter;
    fn into_iter(self) -> CountdownIter {
        CountdownIter(self.0)
    }
}

fn main() {
    // Now this just works!
    for n in Countdown(5) {
        println!("{}...", n);  // 5... 4... 3... 2... 1...
    }
}
```

Once you implement `IntoIterator`, your type works with `for` loops, `.collect()`, and any function that accepts `impl IntoIterator`.

#### Comparison with Other Languages

| Feature | Rust (`IntoIterator`) | Python (`__iter__`) | Java (`Iterable<T>`) |
|--------|----------------------|--------------------|-----------------------|
| Protocol | `into_iter(self) → Iterator` | `__iter__(self) → iterator` | `iterator() → Iterator<T>` |
| Borrowing modes | 3 impls (`&T`, `&mut T`, `T`) | 1 (always reference) | 1 (always reference) |
| Ownership transfer | `for x in v` consumes `v` | Impossible | Impossible |
| Compile-time safety | Yes (borrow checker) | No | No |
| Works with `for` | Automatically via desugaring | Automatically via protocol | Automatically via enhanced for |

The three-borrowing-mode design is unique to Rust. Python and Java have **no concept** of consuming a collection during iteration — their garbage collectors keep everything alive. Rust's `IntoIterator` encodes ownership semantics directly into the iteration protocol, giving you **compile-time guarantees** about whether the collection survives the loop.

---

## Exercises

### Exercise 1: Manual Iterator

Manually call `next()` on different iterators:

```rust
fn main() {
    let mut r = 1..=3;
    println!("{:?}", r.next());  // ?
    println!("{:?}", r.next());  // ?
    println!("{:?}", r.next());  // ?
    println!("{:?}", r.next());  // ?
}
```

<details>
<summary>Answer</summary>

```
Some(1)
Some(2)
Some(3)
None
```

</details>

### Exercise 2: iter vs into_iter

What's the difference in output?

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];
    
    // Version 1
    for s in &v {
        println!("{}", s);
    }
    println!("{:?}", v);  // Does this work?
    
    // Version 2
    for s in v {
        println!("{}", s);
    }
    // println!("{:?}", v);  // Does this work?
}
```

<details>
<summary>Answer</summary>

```
Version 1: &v → v.iter() → s is &String. v is still usable. ✅
Version 2: v → v.into_iter() → s is String (owned). v is consumed. ❌
```

</details>

---

## Summary

| Concept | Detail |
|---------|--------|
| `Iterator` trait | Requires only `next() → Option<Item>` |
| `.iter()` | Borrows: yields `&T` |
| `.iter_mut()` | Mutably borrows: yields `&mut T` |
| `.into_iter()` | Consumes: yields `T` |
| Laziness | Adaptors don't execute until consumed |
| `for x in collection` | Calls `.into_iter()` automatically |
| `IntoIterator` | Trait for anything convertible to an iterator |

---

**Previous:** [← Closures and Ownership](./03-closures-and-ownership.md) · **Next:** [Iterator Adaptors →](./05-iterator-adaptors.md)

<p align="center"><i>Tutorial 4 of 7 — Stage 6: Closures and Iterators</i></p>

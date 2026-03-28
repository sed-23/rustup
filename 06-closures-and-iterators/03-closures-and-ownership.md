# Closures and Ownership 🔑

> **Understanding how closures interact with ownership — capturing by reference vs. by value, the `move` keyword, and lifetimes — is essential for using closures with threads, async code, and returning closures from functions.**

---

## Table of Contents

- [Capture Modes Recap](#capture-modes-recap)
- [The `move` Keyword](#the-move-keyword)
- [Move and Copy Types](#move-and-copy-types)
- [Closures and Threads](#closures-and-threads)
- [Returning Closures](#returning-closures)
- [Closure Size](#closure-size)
- [Partial Captures](#partial-captures)
- [Common Pitfalls](#common-pitfalls)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Capture Modes Recap

Rust automatically chooses the least restrictive capture mode:

```rust
fn main() {
    let s = String::from("hello");
    let mut v = vec![1, 2, 3];
    
    // Capture by &T (immutable reference) — just reads
    let len = || s.len();
    println!("{}", len());
    println!("{}", s);  // ✅ s still available
    
    // Capture by &mut T (mutable reference) — modifies
    let mut push = || v.push(4);
    push();
    // Can't use v here while push holds &mut v
    push();
    println!("{:?}", v);  // ✅ [1, 2, 3, 4, 4] — push no longer borrowed
    
    // Capture by value — consumes
    let consume = || drop(s);  // s moved into closure
    consume();
    // println!("{}", s);  // ❌ s was moved
}
```

---

## The `move` Keyword

`move` forces ALL captures to be by value, even if the closure only reads them:

```rust
fn main() {
    let name = String::from("Alice");
    let age = 30;
    
    // Without move: borrows name
    let greet = || println!("{} is {}", name, age);
    greet();
    println!("{}", name);  // ✅ Still works
    
    // With move: moves name, copies age
    let greet = move || println!("{} is {}", name, age);
    greet();
    // println!("{}", name);  // ❌ name was moved
    println!("{}", age);      // ✅ age was copied (i32 is Copy)
}
```

### When You Need `move`

**1. Returning closures from functions:**

```rust
fn make_greeter(name: String) -> impl Fn() {
    // `name` is a local variable — it will be dropped when function returns
    // The closure MUST own it
    move || println!("Hello, {}!", name)
    // Without move: compile error — closure would hold a dangling reference
}

fn main() {
    let greet = make_greeter(String::from("Alice"));
    greet();  // "Hello, Alice!"
}
```

**2. Sending closures to threads:**

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    
    // Thread might outlive current scope — must move data
    let handle = thread::spawn(move || {
        println!("From thread: {:?}", data);
    });
    
    // println!("{:?}", data);  // ❌ data moved to thread
    handle.join().unwrap();
}
```

**3. Closures that outlive the captured variables:**

```rust
fn main() {
    let closure;
    {
        let name = String::from("Alice");
        closure = move || println!("{}", name);
        // Without move, `name` would be dropped here, leaving a dangling ref
    }
    closure();  // ✅ Works because name was moved into closure
}
```

---

## Move and Copy Types

`move` **copies** types that implement `Copy`, and **moves** types that don't:

```rust
fn main() {
    let x = 42;          // i32: Copy
    let s = String::from("hello");  // String: not Copy
    let b = true;        // bool: Copy
    
    let closure = move || {
        println!("{} {} {}", x, s, b);
    };
    
    // After move:
    println!("{}", x);    // ✅ x was COPIED into closure
    // println!("{}", s); // ❌ s was MOVED into closure
    println!("{}", b);    // ✅ b was COPIED into closure
    
    closure();
}
```

```
move keyword behavior:
┌─────────────┬──────────┬─────────────────────────┐
│ Type        │ Copy?    │ What move does           │
├─────────────┼──────────┼─────────────────────────┤
│ i32, f64    │ Yes      │ Copies value             │
│ bool, char  │ Yes      │ Copies value             │
│ &T          │ Yes      │ Copies the reference     │
│ String      │ No       │ Moves ownership          │
│ Vec<T>      │ No       │ Moves ownership          │
│ Box<T>      │ No       │ Moves ownership          │
└─────────────┴──────────┴─────────────────────────┘
```

---

## Closures and Threads

Threads require `'static` lifetime — closures must own all their data:

```rust
use std::thread;

fn main() {
    // Shared data with Arc (for read-only sharing)
    use std::sync::Arc;
    
    let data = Arc::new(vec![1, 2, 3, 4, 5]);
    let mut handles = vec![];
    
    for i in 0..3 {
        let data = Arc::clone(&data);  // Clone the Arc (cheap)
        handles.push(thread::spawn(move || {
            // Each thread owns its own Arc clone
            let sum: i32 = data.iter().sum();
            println!("Thread {}: sum = {}", i, sum);
        }));
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
}
```

```rust
use std::thread;
use std::sync::{Arc, Mutex};

fn main() {
    // Shared mutable state with Arc<Mutex<T>>
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        }));
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Counter: {}", *counter.lock().unwrap());  // 10
}
```

### The `move` Keyword — Why Threads Need Owned Data

When you call `thread::spawn`, the new thread might run **longer than the function that
created it**. This is the fundamental reason closures sent to threads must own their data.

```
Timeline — why borrowing fails for threads:

  main thread            spawned thread
  ──────────             ──────────────
  let data = vec![…];    
  spawn(|| use data) ──► starts running...
  ← function returns     ...still running...
  ✗ data DROPPED here    ...reads data → 💥 dangling reference!
```

`move` forces the closure to **take ownership** of every captured variable. The data
then lives as long as the closure — which lives as long as the thread.

#### Why `thread::spawn` requires `FnOnce() + Send + 'static`

| Bound      | Meaning                                                      |
|------------|--------------------------------------------------------------|
| `FnOnce()` | The closure can be called (at least once)                    |
| `Send`     | All captured values can safely cross thread boundaries       |
| `'static`  | No borrowed references to **local** data — everything owned  |

The `'static` bound is the key: it forbids closures that borrow local variables,
because locals will be dropped when the scope ends. You can satisfy `'static` in
two ways: **move** owned data in, or only reference `&'static` data (e.g., string
literals or `lazy_static` globals).

#### How other languages solve this

```
┌───────────┬──────────────────────────────────────────────────────────┐
│ Language  │ Strategy                                                 │
├───────────┼──────────────────────────────────────────────────────────┤
│ C / C++   │ No protection. Threads can reference freed stack data.   │
│           │ Use-after-free and data races are common bugs.            │
│ Java      │ GC keeps objects alive while any thread holds a ref.     │
│ Python    │ GIL prevents true parallelism + GC keeps data alive.     │
│ Go        │ Goroutines share the GC heap; races are *possible*.      │
│ Rust      │ Compiler PROVES at compile time: no data race can occur. │
└───────────┴──────────────────────────────────────────────────────────┘
```

The trade-off: sometimes you must `.clone()` data for each thread. That's the
price of safety — a small runtime cost in exchange for a **guarantee** that no
thread will ever read freed memory.

```rust
// Common pattern: clone before spawning
let config = Arc::new(load_config());
for id in 0..4 {
    let config = Arc::clone(&config);   // cheap ref-count bump
    thread::spawn(move || {
        work(id, &config);              // each thread owns an Arc handle
    });
}
// Arc ensures config lives until the last thread finishes.
```

---

## Returning Closures

### With `impl Fn`

```rust
// Returns a single closure type
fn multiplier(factor: i32) -> impl Fn(i32) -> i32 {
    move |x| x * factor
}

fn main() {
    let double = multiplier(2);
    let triple = multiplier(3);
    println!("{}", double(5));   // 10
    println!("{}", triple(5));   // 15
}
```

### With `Box<dyn Fn>` (when return type varies)

```rust
fn make_op(op: char) -> Box<dyn Fn(f64, f64) -> f64> {
    match op {
        '+' => Box::new(|a, b| a + b),
        '-' => Box::new(|a, b| a - b),
        '*' => Box::new(|a, b| a * b),
        '/' => Box::new(|a, b| a / b),
        _ => Box::new(|_, _| f64::NAN),
    }
}

fn main() {
    let add = make_op('+');
    let mul = make_op('*');
    println!("{}", add(3.0, 4.0));  // 7.0
    println!("{}", mul(3.0, 4.0));  // 12.0
}
```

---

### Closures and Lifetimes — The Hidden Dimension

Every closure that borrows data carries an **implicit lifetime** tied to the data it captures.
You rarely see this because the compiler infers it, but when you try to *return* a
borrowing closure, the hidden dimension surfaces.

#### The problem: returning a borrowing closure

```rust
// ❌ Won't compile — closure borrows `data`, which lives only
//    inside this function.
fn make_printer(data: &str) -> impl Fn() {
    || println!("{}", data)   // borrows `data` via &str reference
}
```

The compiler error is revealing:

```text
error: lifetime may not live long enough
  --> src/main.rs:3:5
   |
1  | fn make_printer(data: &str) -> impl Fn() {
   |                       - let's call the lifetime of this reference '1
2  |     || println!("{}", data)
   |     ^^^^^^^^^^^^^^^^^^^^^^^ returning this value requires that '1 must outlive 'static
```

The returned closure borrows something whose lifetime the caller controls.
The type `impl Fn()` has an implicit `'static` requirement — the closure
must live *forever*, but the reference might not.

#### The explicit-lifetime escape hatch: `impl Fn + 'a`

You can tell Rust: "this closure lives at least as long as the data it borrows":

```rust
fn make_printer<'a>(data: &'a str) -> impl Fn() + 'a {
    move || println!("{}", data)   // capture the &str by value (copy of the reference)
}

fn main() {
    let message = String::from("hello");
    let printer = make_printer(&message);
    printer();           // "hello"
    println!("{}", message);  // ✅ message still available
}
```

Here `+ 'a` means: *the closure object itself cannot outlive `'a`*.
Note that `move` copies the `&'a str` (a reference is `Copy`) into the closure;
it does **not** take ownership of the `String`.

#### The easier path: full ownership via `move`

If you don't want to juggle lifetimes, move the data into the closure entirely:

```rust
fn make_printer(data: String) -> impl Fn() {
    move || println!("{}", data)   // data is now owned by the closure
}
```

No lifetime annotation needed — the closure owns everything, so it is `'static`.

#### Mental model

| Return type | Captures | Lifetime concern |
|---|---|---|
| `impl Fn()` | nothing or only owned data | None — implicitly `'static` |
| `impl Fn() + 'a` | borrows data with lifetime `'a` | Caller must keep source alive |
| `Box<dyn Fn() + 'a>` | borrows data, needs dynamic dispatch | Same, but heap-allocated |

Ownership and lifetimes are Rust's **two pillars of safety**. Closures sit right at
the intersection — they can own data (pillar 1) or borrow it (pillar 2). Mastering
returned closures means understanding how both pillars interact.

---

## Closure Size

A closure's size equals the size of everything it captures:

```rust
fn main() {
    let a = 42_i32;       // 4 bytes
    let b = 3.14_f64;     // 8 bytes
    let c = true;         // 1 byte
    
    let closure = || {
        println!("{} {} {}", a, b, c);
    };
    
    println!("Size: {}", std::mem::size_of_val(&closure));
    // Likely 16 bytes (4 + 8 + 1 + padding)
    
    // No captures → zero-sized!
    let empty = || println!("hello");
    println!("Size: {}", std::mem::size_of_val(&empty));  // 0
    
    // Captures a String (24 bytes for ptr+len+cap)
    let s = String::from("hello");
    let with_string = move || println!("{}", s);
    println!("Size: {}", std::mem::size_of_val(&with_string));  // 24
}
```

### Closure Size — How Big Is Your Closure?

Every closure is an anonymous struct whose fields are the captured variables.
Its size is the **sum of all captured values**, subject to alignment padding.

```
Capture                        Size (bytes)  Why
─────────────────────────────  ────────────  ──────────────────────
Nothing (|| println!("hi"))          0       Zero-sized type (ZST)
One i32                              4       Plain integer
One &str                            16       Fat pointer: ptr + len
One String                          24       ptr + len + capacity
One Vec<u8>                         24       ptr + len + capacity
Two i32 values                       8       4 + 4
i32 + String                        32       4 + padding + 24
```

You can always check at runtime:

```rust
let x = 10_i32;
let s = String::from("hello");
let c = move || println!("{x} {s}");
println!("{}", std::mem::size_of_val(&c)); // 32 (with alignment)
```

#### `impl Fn()` vs `Box<dyn Fn()>` — Stack vs Heap

| Approach          | Dispatch   | Allocation | Inlining | Best for                          |
|-------------------|------------|------------|----------|-----------------------------------|
| `impl Fn()`       | Static     | Stack      | ✅ Yes   | Default choice — fast, zero-cost  |
| `&dyn Fn()`       | Dynamic    | Stack ref  | ❌ No    | Borrowing heterogeneous closures  |
| `Box<dyn Fn()>`   | Dynamic    | Heap       | ❌ No    | Storing heterogeneous closures    |

**Monomorphization** with `impl Fn()`: the compiler generates a specialized copy
of the function for *each* closure type. This enables full inlining and
optimization, but increases binary size if you have many distinct closures.

**Dynamic dispatch** with `dyn Fn()`: one version of the function handles all
closures via a vtable. Each call pays ~2 ns of indirection overhead, but the
binary stays smaller.

```
Monomorphized (impl Fn):           Dynamic dispatch (dyn Fn):
┌──────────────────────┐           ┌──────────────────────┐
│ fn apply<C1>(c: C1)  │           │ fn apply(c: &dyn Fn) │
│   → inlined body     │           │   → vtable lookup    │
├──────────────────────┤           │   → indirect call    │
│ fn apply<C2>(c: C2)  │           └──────────────────────┘
│   → inlined body     │            One copy for all types
└──────────────────────┘
 One copy per closure type
```

**Rule of thumb:** use `impl Fn()` (generics) by default. Reach for
`Box<dyn Fn()>` only when you need to store **different** closure types in
the same collection (e.g., a `Vec<Box<dyn Fn()>>` of callbacks).

---

## Partial Captures

Closures can capture individual fields of a struct (since Rust 2021):

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let mut p = Point { x: 10, y: 20 };
    
    // Only captures p.x (not all of p)
    let read_x = || println!("x = {}", p.x);
    
    // Can still mutate p.y while read_x borrows p.x
    p.y = 30;  // ✅ Different field
    
    read_x();
    
    // move keyword captures the ENTIRE struct, not individual fields
    let p = Point { x: 1, y: 2 };
    let closure = move || println!("{} {}", p.x, p.y);
    // p.x and p.y are both moved
}
```

### Partial Captures — Rust Gets Smarter (2021 Edition)

Before Rust 2021, closures captured **entire variables** even if they only used
one field. This caused surprising borrow-checker errors:

```rust
// Pre-2021 behavior (Rust 2018):
struct Pair { a: String, b: String }

let pair = Pair { a: "hello".into(), b: "world".into() };
let c1 = || println!("{}", pair.a);  // captures ALL of `pair`
// let c2 = || println!("{}", pair.b);  // ❌ pair already borrowed by c1!
```

Rust 2021 introduced **disjoint capture** — closures now capture the **exact
fields** they use, not the whole struct:

```rust
// Rust 2021 behavior:
struct Pair { a: String, b: String }

let pair = Pair { a: "hello".into(), b: "world".into() };
let c1 = || println!("{}", pair.a);  // captures only `pair.a`
let c2 = || println!("{}", pair.b);  // captures only `pair.b` ✅
c1();
c2();
```

#### How it works internally

```
Rust 2018 — whole-struct capture:       Rust 2021 — disjoint capture:
┌────────────────────────────┐          ┌────────────────────────────┐
│ Anonymous struct for c1    │          │ Anonymous struct for c1    │
│ ┌────────────────────────┐ │          │ ┌────────────────────────┐ │
│ │ pair: &Pair            │ │          │ │ pair_a: &String        │ │
│ │   (entire struct)      │ │          │ │   (only field a)       │ │
│ └────────────────────────┘ │          │ └────────────────────────┘ │
└────────────────────────────┘          └────────────────────────────┘
  → blocks ALL other borrows             → field b is free to borrow
```

#### Migrating from 2018 to 2021

When running `cargo fix --edition`, Rust automatically inserts dummy
captures where full-struct semantics were relied on:

```rust
// Auto-inserted by cargo fix:
let _ = &pair;  // force whole-struct capture for backward compat
let c = || println!("{}", pair.a);
```

This keeps old code working while new code benefits from disjoint capture.

#### Why this matters

This was one of the biggest quality-of-life improvements in the 2021 edition.
It reflects Rust's philosophy: **make the compiler smarter so the programmer
writes natural code**. Instead of requiring workarounds like:

```rust
// Old workaround: destructure to get separate bindings
let Pair { a, b } = pair;
let c1 = || println!("{}", a);
let c2 = || println!("{}", b);
```

…you now just write the obvious code and the compiler does the right thing.

---

### Closure Capture Optimization — What the Compiler Really Does

When the Rust compiler encounters a closure, it performs **capture analysis**
during type checking. Here's the full pipeline:

#### Step 1 — Determine minimum capture mode per variable

For each captured variable (or field, in Rust 2021), the compiler picks the
least powerful mode that satisfies every usage inside the closure body:

| Priority | Mode | When chosen |
|----------|------|-------------|
| 1 (cheapest) | `&T` | Closure only reads the value |
| 2 | `&mut T` | Closure mutates the value |
| 3 (most expensive) | `T` (by value) | Closure drops, moves, or returns the value |

In Rust 2021 this analysis happens at the **field level**, not the struct level.
That's the "partial captures" feature in action.

#### Step 2 — Generate a unique anonymous type

The compiler creates a **unique, unnameable struct** for each closure expression.
Its fields are the captured variables (in the mode determined above).

```rust
let x = 10;
let y = String::from("hi");
let c = || println!("{x} {y}");
// Compiler generates (conceptually):
// struct __closure_c<'a> {
//     x: &'a i32,
//     y: &'a String,
// }
// impl<'a> Fn<()> for __closure_c<'a> { ... }
```

> **Fun fact:** each closure has a *unique* type even if the code looks
> identical. Two `|x| x + 1` closures written on different lines are
> **different types**. This is why you can't put heterogeneous closures in a
> `Vec<impl Fn>` — you need `Vec<Box<dyn Fn(i32) -> i32>>` instead.

#### Step 3 — Implement the right `Fn*` trait(s)

Based on the capture modes the compiler now implements one or more of
`Fn`, `FnMut`, or `FnOnce`:

```text
Captures only &T   → implements Fn (and FnMut, FnOnce)
Captures &mut T    → implements FnMut (and FnOnce)
Captures T (move)  → implements FnOnce
```

#### LLVM-level optimizations

Once MIR (Mid-level Intermediate Representation) is lowered to LLVM IR, two
key optimizations kick in:

1. **Zero-capture closures → function pointers.** If a closure captures
   nothing, the compiler can coerce it to a plain `fn()` pointer. This is why
   `let f: fn(i32) -> i32 = |x| x + 1;` compiles.

2. **Inlining through generics.** When a function takes `impl Fn(i32) -> i32`,
   the compiler monomorphises it for each unique closure type. Combined with
   the `#[inline]` attribute, LLVM can inline the closure body directly into
   the call site, yielding **zero-cost abstraction** — literally no overhead
   compared to hand-written loops.

```rust
// This generic function will be monomorphised for each closure passed to it.
// LLVM can then inline the closure body entirely.
#[inline]
fn apply_twice(f: impl Fn(i32) -> i32, x: i32) -> i32 {
    f(f(x))
}

fn main() {
    // After optimisation this compiles to the same machine code as:
    // let result = (5 + 1) + 1;
    let result = apply_twice(|x| x + 1, 5);
    println!("{result}");  // 7
}
```

This is why idiomatic Rust iterator chains (`.map().filter().collect()`) are
as fast as hand-rolled `for` loops — each closure is monomorphised and inlined.

---

## Common Pitfalls

### Pitfall 1: Borrowing conflicts

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    
    // Closure borrows data mutably
    let mut push = || data.push(4);
    
    // Can't use data while closure holds &mut
    // println!("{:?}", data);  // ❌
    // data.len();               // ❌
    
    push();
    println!("{:?}", data);  // ✅ After last use of push
}
```

### Pitfall 2: Moving in a loop

```rust
fn main() {
    let name = String::from("Alice");
    
    // ❌ Can't move same value multiple times
    // for _ in 0..3 {
    //     let c = move || println!("{}", name);  // name moved first iteration
    //     c();
    // }
    
    // ✅ Clone for each iteration
    for _ in 0..3 {
        let name = name.clone();
        let c = move || println!("{}", name);
        c();
    }
    
    // ✅ Or just borrow
    for _ in 0..3 {
        let c = || println!("{}", name);  // borrows, doesn't move
        c();
    }
}
```

### Pitfall 3: Returning closures without move

```rust
// ❌ This won't compile
// fn bad() -> impl Fn() {
//     let s = String::from("hello");
//     || println!("{}", s)  // s is borrowed but about to be dropped!
// }

// ✅ Use move
fn good() -> impl Fn() {
    let s = String::from("hello");
    move || println!("{}", s)  // s is moved into closure
}
```

---

### Closures in Real Rust Codebases — Production Patterns

Closures in Rust are not just "lambda functions" — they are a fundamental
building block of the language's API design philosophy. Here are five patterns
you'll encounter in virtually every production codebase.

#### Pattern 1: Builder pattern with closures

Builders often accept closures for configuring callbacks:

```rust
struct ServerConfig {
    on_error: Box<dyn Fn(&str)>,
    port: u16,
}

impl ServerConfig {
    fn new() -> Self {
        Self { on_error: Box::new(|_| {}), port: 8080 }
    }
    fn on_error(mut self, handler: impl Fn(&str) + 'static) -> Self {
        self.on_error = Box::new(handler);
        self
    }
    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }
}

fn main() {
    let config = ServerConfig::new()
        .port(3000)
        .on_error(|e| eprintln!("ERROR: {e}"));
    
    (config.on_error)("disk full");  // ERROR: disk full
}
```

#### Pattern 2: Dependency injection with closures

Instead of defining a trait and a mock struct, pass behaviour as a closure:

```rust
fn fetch_and_process(
    fetcher: impl Fn(&str) -> Result<String, String>,
) -> Result<usize, String> {
    let body = fetcher("https://example.com/api")?;
    Ok(body.len())
}

// Production: real HTTP call
// let result = fetch_and_process(|url| http::get(url));

// Test: deterministic fake
#[test]
fn test_fetch() {
    let result = fetch_and_process(|_url| Ok("hello".to_string()));
    assert_eq!(result.unwrap(), 5);
}
```

#### Pattern 3: Custom iterator combinators

You can extend `Iterator` with your own methods that accept closures:

```rust
trait IteratorExt: Iterator + Sized {
    /// Keep only elements where the predicate returns `true`,
    /// but also count how many were discarded.
    fn filter_counted<P>(self, pred: P) -> (Vec<Self::Item>, usize)
    where
        P: Fn(&Self::Item) -> bool,
    {
        let mut kept = Vec::new();
        let mut discarded = 0;
        for item in self {
            if pred(&item) { kept.push(item); } else { discarded += 1; }
        }
        (kept, discarded)
    }
}
impl<I: Iterator> IteratorExt for I {}

fn main() {
    let (evens, dropped) = (0..10).filter_counted(|x| x % 2 == 0);
    println!("{evens:?}, dropped {dropped}");  // [0, 2, 4, 6, 8], dropped 5
}
```

#### Pattern 4: Error handling with closures

`unwrap_or_else` defers the computation of the fallback value:

```rust
fn get_config_value(key: &str) -> Option<String> {
    std::env::var(key).ok()
}

fn main() {
    // The closure only runs if the Option is None — lazy evaluation.
    let db_url = get_config_value("DATABASE_URL")
        .unwrap_or_else(|| {
            eprintln!("DATABASE_URL not set, using default");
            "postgres://localhost/dev".to_string()
        });
    println!("connecting to {db_url}");
}
```

#### Pattern 5: Testing with closures

Pass closures as mock dependencies so tests stay fast and deterministic:

```rust
fn run_with_clock(
    now: impl Fn() -> u64,
    threshold: u64,
) -> &'static str {
    if now() > threshold { "expired" } else { "valid" }
}

#[test]
fn test_expired() {
    assert_eq!(run_with_clock(|| 1000, 500), "expired");
}

#[test]
fn test_valid() {
    assert_eq!(run_with_clock(|| 100, 500), "valid");
}
```

#### The takeaway

| Pattern | Why closures shine |
|---|---|
| Builder callbacks | Capture user context without extra structs |
| Dependency injection | Swap behaviour in tests with zero boilerplate |
| Custom combinators | Compose operations naturally in iterator chains |
| Lazy error handling | Compute fallback only when needed (`unwrap_or_else`) |
| Test mocks | Replace real I/O with one-line lambdas |

Closures are Rust's answer to the *expression problem*: they let library authors
expose hook points, and library consumers plug in behavior — all with full type
safety and zero runtime cost when using generics.

---

## Exercises

### Exercise 1: Closure Capture Analysis

What does each closure capture, and by what mode?

```rust
fn main() {
    let a = String::from("hello");
    let mut b = vec![1, 2, 3];
    let c = 42;
    
    let c1 = || println!("{}", a);          // ?
    let c2 = || { b.push(4); };            // ?
    let c3 = move || println!("{} {}", a, c); // ?
}
```

<details>
<summary>Answer</summary>

```
c1: captures `a` by &String (immutable reference, Fn)
c2: captures `b` by &mut Vec<i32> (mutable reference, FnMut)
c3: captures `a` by value (moved), `c` by value (copied since i32: Copy)
    Note: c1 and c3 conflict — can't have both (a borrowed by c1, moved by c3)
    This wouldn't actually compile as written.
```

</details>

### Exercise 2: Thread-Safe Counter

Create a function that returns a thread-safe incrementing closure:

<details>
<summary>Solution</summary>

```rust
use std::sync::{Arc, Mutex};

fn make_counter() -> impl FnMut() -> i32 {
    let count = Arc::new(Mutex::new(0));
    let count_clone = Arc::clone(&count);
    
    move || {
        let mut num = count_clone.lock().unwrap();
        *num += 1;
        *num
    }
}

fn main() {
    let mut counter = make_counter();
    println!("{}", counter());  // 1
    println!("{}", counter());  // 2
    println!("{}", counter());  // 3
}
```

</details>

---

## Summary

| Concept | Detail |
|---------|--------|
| Default capture | Least restrictive: `&T` → `&mut T` → by value |
| `move` keyword | Forces all captures by value |
| Copy types + move | Value is copied (still usable after) |
| Non-Copy + move | Value is moved (no longer usable) |
| Threads | Require `move` closures (`'static` lifetime) |
| Return closures | Need `move` to avoid dangling references |
| Closure size | Sum of captured values' sizes |
| Partial captures | Rust 2021 can capture individual struct fields |

---

**Previous:** [← Closure Traits](./02-closure-traits.md) · **Next:** [The Iterator Trait →](./04-iterator-trait.md)

<p align="center"><i>Tutorial 3 of 7 — Stage 6: Closures and Iterators</i></p>

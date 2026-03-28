# Closures 🔒

> **Closures are anonymous functions that can capture variables from their surrounding scope. They're the foundation of functional-style programming in Rust — used everywhere with iterators, callbacks, and higher-order functions.**

---

## Table of Contents

- [What Is a Closure?](#what-is-a-closure)
- [Closure Syntax](#closure-syntax)
- [Type Inference](#type-inference)
- [Capturing Variables](#capturing-variables)
- [Closures as Parameters](#closures-as-parameters)
- [Closures as Return Values](#closures-as-return-values)
- [Closures vs Functions](#closures-vs-functions)
- [Common Usage Patterns](#common-usage-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is a Closure?

A closure is an anonymous function that can "close over" (capture) variables from the scope where it's defined:

```rust
fn main() {
    let name = String::from("Alice");
    
    // A closure that captures `name`
    let greet = || {
        println!("Hello, {}!", name);  // Uses `name` from outer scope
    };
    
    greet();  // "Hello, Alice!"
    greet();  // Can call multiple times
}
```

Compare with a regular function:

```rust
// Functions can't capture local variables
fn greet() {
    // println!("{}", name);  // ❌ `name` not in scope
}
```

---

### The History of Closures — From Lambda Calculus to Rust

Closures didn't just appear in Rust out of thin air. They have a nearly 90-year intellectual
lineage stretching from pure mathematics to every modern programming language.

**Lambda Calculus (1930s)** — Alonzo Church developed lambda calculus as a formal system
for expressing computation using functions alone. The key insight: functions are *values*.
You can pass them around, return them, and nest them. This was pure math — no computers
existed yet — but it laid the theoretical foundation for every closure in every language.

```
λx. x + 1          ← a function that adds 1 (Church's notation)
|x| x + 1          ← the same idea in Rust, ~90 years later
```

**LISP (1958)** — John McCarthy created LISP, the first programming language to treat
functions as first-class values. LISP's `lambda` expressions could reference variables from
enclosing scopes, making it the first language with closures in practice — though the
term "closure" wasn't coined until later.

**Scheme (1975)** — Scheme (a LISP dialect) popularized *lexical scoping* — the rule that
a function sees variables from where it was *defined*, not where it's *called*. This is
the scoping rule Rust (and most modern languages) uses.

**JavaScript (1995)** — Closures entered the mainstream through the web. Every JS function
is a closure. However, `var` hoisting and lack of block scoping led to infamous bugs —
the classic `for` loop closure problem that confused a generation of developers.

**C++ (2011)** — Lambdas added with explicit capture lists: `[x, &y](int z) { return x + y + z; }`.
Programmers specify *how* each variable is captured — by value or by reference. Rust's
approach is spiritually similar but the compiler decides the capture mode automatically.

**Java (2014)** — Lambdas arrived in Java 8, but with a restriction: captured variables
must be "effectively final" — you cannot mutate them. This sidesteps data-race concerns
but limits expressiveness compared to Rust's `FnMut`.

**Rust (2015)** — Rust's closures capture using the *least restrictive mode needed*:
`&T` if the closure only reads, `&mut T` if it mutates, or `T` (move) if it consumes.
The ownership system makes closures both powerful and memory-safe — no garbage collector
required.

```
Timeline of Closures
─────────────────────────────────────────────────────────────
1930s       1958      1975      1995    2011  2014  2015
  │           │         │         │       │     │     │
  λ-calc     LISP    Scheme      JS     C++  Java  Rust
 (Church)  (McCarthy) (lexical  (main-  (cap- (eff. (owner-
                      scoping)  stream) ture   final) ship-
                                        lists)       aware)
─────────────────────────────────────────────────────────────
```

| Language | Closure Syntax | Capture Mode | Can Mutate Captures? | Since |
|----------|---------------|-------------|---------------------|-------|
| LISP | `(lambda (x) (+ x y))` | Implicit (reference) | Yes | 1958 |
| Scheme | `(lambda (x) (+ x y))` | Lexical scoping | Yes | 1975 |
| JavaScript | `(x) => x + y` | Implicit (reference) | Yes | 1995 |
| Python | `lambda x: x + y` | Implicit (reference) | Read-only in lambdas | 1994 |
| C++ | `[x, &y](int z){ ... }` | Explicit (`=`, `&`) | If captured by ref | 2011 |
| Java | `(x) -> x + y` | Effectively final | No | 2014 |
| Swift | `{ x in x + y }` | Implicit (reference) | Yes (reference types) | 2014 |
| **Rust** | `\|x\| x + y` | **Auto: `&T`/`&mut T`/`T`** | **Yes (`FnMut`)** | **2015** |

The evolution: from theoretical concept (1930s) → niche academic feature (1958–1990s) →
essential tool in every modern language. Rust's contribution is making closures both
*zero-cost* and *ownership-aware* — a combination no prior language achieved.

---

## Closure Syntax

```rust
fn main() {
    // Full syntax
    let add = |x: i32, y: i32| -> i32 {
        x + y
    };
    
    // Type annotations are optional (inferred)
    let add = |x, y| x + y;
    
    // No parameters
    let say_hi = || println!("Hi!");
    
    // Single expression (no braces needed)
    let double = |x| x * 2;
    
    // Multi-line with braces
    let complex = |x: i32| {
        let squared = x * x;
        let cubed = squared * x;
        cubed + squared + x
    };
    
    println!("{}", add(3, 4));     // 7
    say_hi();                       // Hi!
    println!("{}", double(5));      // 10
    println!("{}", complex(3));     // 39
}
```

Syntax comparison:

```
Function:  fn  add(x: i32, y: i32) -> i32 { x + y }
Closure:       |x: i32, y: i32| -> i32     { x + y }
Closure:       |x, y|                        x + y     (minimal)
```

---

## Type Inference

Closures infer their parameter and return types from usage:

```rust
fn main() {
    let add = |x, y| x + y;
    
    // First call fixes the types
    let result: i32 = add(1, 2);  // x and y are now i32
    
    // Can't use with different types afterward
    // let result: f64 = add(1.0, 2.0);  // ❌ Expected i32, found f64
}
```

Each closure has a **unique, anonymous type** — even two identical closures have different types:

```rust
fn main() {
    let a = |x: i32| x + 1;
    let b = |x: i32| x + 1;
    
    // a and b have DIFFERENT types, even though they look identical
    // let c = if true { a } else { b };  // ❌ Different types
}
```

---

## Capturing Variables

Closures capture variables in three ways (Rust picks the least restrictive):

### 1. By Immutable Reference (`&T`)

```rust
fn main() {
    let name = String::from("Alice");
    
    let greet = || println!("Hello, {}!", name);  // Borrows &name
    
    greet();
    greet();
    println!("{}", name);  // ✅ name still usable (only borrowed)
}
```

### 2. By Mutable Reference (`&mut T`)

```rust
fn main() {
    let mut count = 0;
    
    let mut increment = || {
        count += 1;  // Borrows &mut count
    };
    
    increment();
    increment();
    // Can't use count while increment exists with &mut borrow
    // println!("{}", count);  // ❌ if increment is used after
    
    increment();
    println!("{}", count);  // ✅ 3 (increment not used after this)
}
```

### 3. By Value (move)

```rust
fn main() {
    let name = String::from("Alice");
    
    let greet = move || {
        println!("Hello, {}!", name);  // name MOVED into closure
    };
    
    greet();
    // println!("{}", name);  // ❌ name was moved
}
```

### How Rust Decides

```
Does the closure...
├─ Only READ the variable?      → capture by &T
├─ MUTATE the variable?         → capture by &mut T
└─ MOVE the variable (consume)? → capture by value

The `move` keyword forces ALL captures by value.
```

```rust
fn main() {
    let s = String::from("hello");
    let n = 42;
    
    // Without move: s borrowed, n copied (i32 is Copy)
    let c1 = || println!("{} {}", s, n);
    c1();
    println!("{}", s);  // ✅ s still alive
    
    // With move: s moved, n copied
    let c2 = move || println!("{} {}", s, n);
    c2();
    // println!("{}", s);  // ❌ s was moved into c2
    println!("{}", n);     // ✅ n was copied (i32 is Copy)
}
```

---

### How Closures Are Represented in Memory

Understanding what closures *actually are* in memory removes all the mystery. A closure
is not magic — it's a **compiler-generated anonymous struct** whose fields are the
captured variables.

```rust
// What you write:
let x = 5;
let closure = |y| x + y;

// What the compiler generates (conceptually):
struct Closure_1 {
    x: i32,   // captured variable becomes a struct field
}

impl Fn(i32) -> i32 for Closure_1 {
    fn call(&self, y: i32) -> i32 {
        self.x + y
    }
}

let closure = Closure_1 { x: 5 };
```

**The size of a closure = the sum of its captured variable sizes.** You can verify this
with `std::mem::size_of_val`:

```rust
fn main() {
    let a: i32 = 10;             // 4 bytes
    let b: f64 = 3.14;           // 8 bytes
    let s = String::from("hi");  // 24 bytes (ptr + len + cap)

    let c1 = || println!("{}", a);            // captures &a
    let c2 = || println!("{} {}", a, b);      // captures &a, &b
    let c3 = move || println!("{}", s);        // captures s by value
    let c4 = || 42;                             // captures NOTHING

    println!("c1: {} bytes", std::mem::size_of_val(&c1));  // 8  (one &i32 ref)
    println!("c2: {} bytes", std::mem::size_of_val(&c2));  // 16 (two refs)
    println!("c3: {} bytes", std::mem::size_of_val(&c3));  // 24 (String moved in)
    println!("c4: {} bytes", std::mem::size_of_val(&c4));  // 0  (nothing captured!)
}
```

**Zero-capture closures have size 0.** They contain no data — they're pure code,
indistinguishable from function pointers. That's why Rust lets you coerce them to `fn`
pointer types.

```
Closure Memory Layout
═══════════════════════════════════════════════════════

  |y| x + y        (captures x: i32)
  ┌─────────────┐
  │ x: i32 = 5  │   size = 4 bytes (lives on the STACK)
  └─────────────┘

  || 42             (captures nothing)
  ┌─────────────┐
  │  (empty)    │   size = 0 bytes (zero-sized type!)
  └─────────────┘

  move || s.len()   (captures s: String)
  ┌─────────────────────────┐
  │ ptr: *const u8          │
  │ len: usize              │   size = 24 bytes
  │ cap: usize              │
  └─────────────────────────┘
═══════════════════════════════════════════════════════
```

**How other languages compare:**

| Language | What a Closure Captures | Memory Impact |
|----------|------------------------|---------------|
| **JavaScript** | Reference to the *entire* enclosing scope chain | Keeps all variables in scope alive via GC — even ones the closure doesn't use. Can cause memory leaks. |
| **Python** | Reference to enclosing scope's `__dict__` | Similar to JS — the whole scope stays alive. |
| **Java** | Individual variables (must be effectively final) | Like Rust — only captures what's needed. But always by copy, never by reference. |
| **C++** | Explicit capture list (`[x, &y]`) | Programmer controls layout — similar efficiency to Rust, but manual. |
| **Rust** | Only the variables actually used, by the least restrictive mode | Minimal capture, no GC, stack-allocated. Optimal. |

**Why Rust's approach is efficient:** Closure structs live on the **stack** by default —
no `malloc`, no garbage collector, no reference counting. The compiler knows the exact
size at compile time, so it can inline the closure and often eliminate the struct entirely.
The generated machine code is identical to what you'd write by hand with a plain struct
and a method. This is what "zero-cost abstraction" truly means.

---

## Closures as Parameters

Use generic trait bounds to accept closures:

```rust
// Accept any closure that takes i32 and returns i32
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

// Alternative syntax with `where`
fn apply_where<F>(f: F, x: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    f(x)
}

fn main() {
    let double = |x| x * 2;
    let square = |x| x * x;
    
    println!("{}", apply(double, 5));   // 10
    println!("{}", apply(square, 5));   // 25
    
    // Can also pass regular functions!
    fn triple(x: i32) -> i32 { x * 3 }
    println!("{}", apply(triple, 5));   // 15
}
```

---

### Closures in Functional Programming — map, filter, reduce

The operations `map`, `filter`, and `reduce` (called `fold` in Rust) are the *holy trinity*
of functional programming. They were invented by John McCarthy in **LISP (1958)** and have
since spread to virtually every programming language. Closures are the glue that makes
them work — you pass a closure that defines *what* to do, and the operation handles *how*.

**The same pipeline in four languages — filter even numbers, then sum them:**

```rust
// Rust
let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
let sum: i32 = nums.iter()
    .filter(|&&x| x % 2 == 0)
    .sum();
// sum = 30
```

```python
# Python
nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
total = sum(filter(lambda x: x % 2 == 0, nums))
# total = 30
```

```javascript
// JavaScript
const nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
const total = nums
    .filter(x => x % 2 === 0)
    .reduce((acc, x) => acc + x, 0);
// total = 30
```

```java
// Java (Stream API, since Java 8)
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
int total = nums.stream()
    .filter(x -> x % 2 == 0)
    .reduce(0, Integer::sum);
// total = 30
```

| Operation | Rust | Python | JavaScript | Haskell |
|-----------|------|--------|------------|---------|
| Transform each element | `.map(\|x\| ...)` | `map(lambda x: ..., xs)` | `.map(x => ...)` | `map f xs` |
| Keep elements matching predicate | `.filter(\|x\| ...)` | `filter(lambda x: ..., xs)` | `.filter(x => ...)` | `filter p xs` |
| Accumulate into single value | `.fold(init, \|acc, x\| ...)` | `functools.reduce(lambda a, x: ..., xs)` | `.reduce((a, x) => ..., init)` | `foldl f z xs` |

**Why Rust's version is as fast as a hand-written `for` loop:**

In JavaScript or Python, each `.map()` or `.filter()` creates an intermediate
collection — an entirely new array. Chaining three operations means three arrays.

Rust's iterators are **lazy** — `.filter()` and `.map()` don't produce intermediate
collections. They return *adaptor* structs that process one element at a time. The
compiler then **inlines** the closure and the iterator machinery, producing a single
loop with no allocations:

```rust
// This iterator pipeline:
let sum: i32 = nums.iter().filter(|&&x| x % 2 == 0).sum();

// Compiles to roughly the same machine code as:
let mut sum = 0;
for x in &nums {
    if *x % 2 == 0 { sum += *x; }
}
```

This is the **iterator pipeline** pattern: chain adaptors (`.map`, `.filter`,
`.take`, `.skip`, `.enumerate`, `.zip`, ...) and end with a **consumer**
(`.collect()`, `.sum()`, `.fold()`, `.count()`, `.for_each()`). The consumer
drives the iteration — nothing happens until you call one.

We cover iterators in depth in the next tutorials, but the key takeaway here is:
**closures are the glue** that makes functional pipelines possible. Without closures,
you'd need to write a separate named function for every transformation — closures
make the code concise *and* fast.

---

## Closures as Return Values

Must use `impl Fn` or `Box<dyn Fn>`:

```rust
// Return a closure using impl trait
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n  // Must use `move` to own captured `n`
}

// Return different closures using Box (trait object)
fn make_operation(op: &str) -> Box<dyn Fn(i32, i32) -> i32> {
    match op {
        "add" => Box::new(|a, b| a + b),
        "mul" => Box::new(|a, b| a * b),
        _ => Box::new(|a, b| a - b),
    }
}

fn main() {
    let add5 = make_adder(5);
    println!("{}", add5(10));  // 15
    println!("{}", add5(20));  // 25
    
    let op = make_operation("add");
    println!("{}", op(3, 4));  // 7
}
```

---

## Closures vs Functions

| Feature | Function | Closure |
|---------|----------|---------|
| Named | Yes (`fn name`) | No (anonymous) |
| Captures environment | No | Yes |
| Type annotations | Required | Optional (inferred) |
| Type | Named type (`fn(i32) -> i32`) | Anonymous unique type |
| Can be function pointer | Yes | Only if no captures |

```rust
fn main() {
    // Function pointer type
    let f: fn(i32) -> i32 = |x| x + 1;  // ✅ No captures → can be fn pointer
    
    let n = 5;
    // let f: fn(i32) -> i32 = |x| x + n;  // ❌ Has captures
    let f: Box<dyn Fn(i32) -> i32> = Box::new(|x| x + n);  // ✅ Trait object
}
```

---

### Closures vs Functions — The Compiler's Perspective

To really understand closures, you need to see what the compiler *actually does* with them.
The difference between a function and a closure is structural — it's about data layout in memory.

**A function** is just a code address. It lives in the `.text` segment of your binary.
It has no associated data — every invocation uses the same global pointer.

**A closure** is a *struct* that bundles captured variables together with a function.
When you write a closure, the compiler generates an anonymous struct behind the scenes:

```rust
// What you write:
let offset = 10;
let add_offset = |x: i32| x + offset;

// What the compiler conceptually generates:
struct __ClosureAdd { offset: i32 }  // captured variable becomes a field

impl Fn(i32) -> i32 for __ClosureAdd {
    fn call(&self, x: i32) -> i32 {
        x + self.offset       // closure body becomes the call() method
    }
}

let add_offset = __ClosureAdd { offset: 10 };
```

This generated struct implements one of the `Fn` / `FnMut` / `FnOnce` traits depending
on how the closure uses its captures.

```
Memory Layout Comparison
════════════════════════════════════════════════════════

  Regular function            Closure
  ──────────────────          ──────────────────────
                              ┌──────────────────┐
  Code address: 0x401000      │ offset: 10 (i32) │ ← captured data
      │                       │                  │
      ▼                       │ call(self, x)────│──▶ generated fn
   fn add(x) { x + ??? }     └──────────────────┘
      ↑                             ↑
   No environment!            Has its own environment

════════════════════════════════════════════════════════
```

**Every closure gets a UNIQUE type.** Even two closures with identical code have different types:

```rust
let a = |x: i32| x + 1;   // type: __Closure_A
let b = |x: i32| x + 1;   // type: __Closure_B  (different!)

// This won't compile:
// let funcs: Vec<_> = vec![a, b];  // ❌ mismatched types

// Use trait objects to erase the type:
let funcs: Vec<Box<dyn Fn(i32) -> i32>> = vec![Box::new(a), Box::new(b)];  // ✅
```

Why unique types? Because each closure struct has different fields (different captured data).
This lets the compiler **inline everything** — no virtual dispatch, no heap allocation,
no runtime overhead. This is Rust's "zero-cost abstraction" for closures.

**The function pointer coercion:** If a closure captures *nothing*, its struct is empty —
it's equivalent to a plain function. Rust lets you coerce it to a `fn` pointer:

```rust
let no_capture = |x: i32| x * 2;
let fptr: fn(i32) -> i32 = no_capture;  // ✅ zero-size closure → fn pointer

let n = 5;
let has_capture = |x: i32| x * n;
// let fptr: fn(i32) -> i32 = has_capture;  // ❌ closure has data
```

| Aspect | Function (`fn`) | Closure (no captures) | Closure (with captures) |
|--------|----------------|----------------------|------------------------|
| Memory | Code pointer only | Zero-size struct | Struct with fields |
| Dispatch | Direct call | Direct call (inlined) | Direct call (inlined) |
| Coerce to `fn` ptr | Yes | Yes | No |
| Dynamic dispatch | N/A | Via `Box<dyn Fn>` | Via `Box<dyn Fn>` |
| Runtime cost | Zero | Zero | Zero (stack) / Heap (`Box`) |

The takeaway: closures in Rust *look* like a convenience feature, but under the hood
they compile to the same machine code you'd write by hand with structs and methods.

---

## Common Usage Patterns

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    // Filter
    let evens: Vec<&i32> = nums.iter().filter(|&&x| x % 2 == 0).collect();
    println!("{:?}", evens);  // [2, 4, 6, 8, 10]
    
    // Map
    let squares: Vec<i32> = nums.iter().map(|&x| x * x).collect();
    println!("{:?}", squares);
    
    // Sort with custom comparator
    let mut words = vec!["banana", "apple", "cherry"];
    words.sort_by(|a, b| a.len().cmp(&b.len()));
    println!("{:?}", words);  // ["apple", "banana", "cherry"]
    
    // Callbacks
    fn on_event(event: &str, callback: impl Fn(&str)) {
        println!("Event: {}", event);
        callback(event);
    }
    
    on_event("click", |e| println!("Handled: {}", e));
    
    // Lazy initialization
    let value = Some(42).unwrap_or_else(|| {
        println!("Computing default...");
        expensive_default()
    });
}

fn expensive_default() -> i32 { 0 }
```

---

### Closures in the Real World — Patterns You'll See Everywhere

Closures aren't just a language feature — they're a *design tool*. Here are the patterns
you'll encounter constantly in real Rust codebases.

**Pattern 1: Callbacks — Event-Driven Programming**

Any time you need "do X when Y happens," you reach for a closure:

```rust
button.on_click(|| update_ui());
server.on_request(|req| handle(req));
timer.on_tick(move || counter.fetch_add(1, Ordering::SeqCst));
```

**Pattern 2: The Strategy Pattern — Customizable Behavior**

Instead of inheritance hierarchies, pass a closure to change behavior:

```rust
fn sort_by_key<T, K: Ord>(data: &mut [T], key: impl Fn(&T) -> K) {
    data.sort_by(|a, b| key(a).cmp(&key(b)));
}

sort_by_key(&mut employees, |e| e.salary);        // sort by salary
sort_by_key(&mut employees, |e| e.name.clone());   // sort by name
```

**Pattern 3: Lazy Initialization — Compute Only When Needed**

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| {
        // This closure runs ONCE, only on first access
        load_config_from_disk()
    })
}
```

**Pattern 4: Builder Configuration — Fluent APIs**

```rust
let client = HttpClient::builder()
    .with_timeout(Duration::from_secs(30))
    .on_error(|e| eprintln!("Request failed: {e}"))
    .on_retry(|attempt| println!("Retry #{attempt}"))
    .build();
```

**Pattern 5: Map / Filter / Reduce — Data Pipelines**

```rust
let report: Vec<String> = orders
    .iter()
    .filter(|o| o.total > 100.0)           // keep expensive orders
    .map(|o| format!("{}: ${:.2}", o.id, o.total))  // format
    .collect();
```

**In Web Frameworks (Axum, Actix)** — Route handlers are closures
capturing shared application state:

```rust
// Axum example
let app_state = Arc::new(AppState::new());

let app = Router::new()
    .route("/users", get({
        let state = app_state.clone();
        move || async move { list_users(&state).await }
    }));
```

**In Async Rust** — Every `async { }` block is essentially a closure
that returns a `Future`. The same capture rules apply:

```rust
let data = fetch_data().await;
tokio::spawn(async move {
    // `data` moved into this async block (closure-like capture)
    process(data).await;
});
```

```
Where Closures Appear in a Typical Rust Project
───────────────────────────────────────────────────
  Iterator chains          ████████████████  40%
  Callbacks / handlers     ██████████        25%
  Async blocks / spawns    ████████          20%
  Builder / config APIs    ████              10%
  Other (sort, test, etc)  ██                 5%
───────────────────────────────────────────────────
```

> **The key insight:** In Rust, closures are so efficient that using them is **never**
> slower than writing the equivalent code inline. The compiler monomorphizes each
> closure, inlines the call, and often produces *identical* machine code. Use closures
> freely — they are a zero-cost abstraction in the truest sense.

---

### Closures and Async — The Hidden Connection

If you plan to do *anything* with async Rust — web servers, network clients, concurrent
task processing — you need to understand this: **async blocks are closures in disguise.**

When you write `async { ... }`, the compiler generates an anonymous type that implements
the `Future` trait — just like a closure generates an anonymous type implementing `Fn`.
The `async` block captures variables from its environment using the *exact same rules*
as closures: by reference if it only reads, by mutable reference if it mutates, by value
if you use `move`.

```rust
// A closure that captures `name`:
let name = String::from("Alice");
let greet = move || println!("Hello, {name}");

// An async block that captures `name` — same `move` keyword, same rules:
let name = String::from("Alice");
let future = async move { println!("Hello, {name}") };
```

**Spawning tasks requires `move` for the same reason closures do** — the task might
outlive the current scope, so it must *own* everything it uses:

```rust
use tokio;

#[tokio::main]
async fn main() {
    let data = vec![1, 2, 3];

    // `move` transfers ownership of `data` into the spawned task
    tokio::spawn(async move {
        println!("Processing {:?}", data);
    });
}
```

Without `move`, the compiler would reject this — `data` is a local variable that could
be dropped before the spawned task runs. Exactly the same lifetime reasoning as closures.

**Web frameworks are built on closures.** Here's how Axum and Actix use them:

```rust
// Axum — route handlers are closures (or functions) returning Futures
use axum::{Router, routing::get};

let app = Router::new()
    .route("/", get(|| async { "Hello, World!" }))
    .route("/greet/:name", get(|path: axum::extract::Path<String>| async move {
        format!("Hello, {}!", path.0)
    }));
```

```rust
// Actix-web — same pattern, closures everywhere
use actix_web::{web, App, HttpServer};

HttpServer::new(|| {
    App::new()
        .route("/", web::get().to(|| async { "Hello!" }))
})
```

Notice the nesting: `HttpServer::new` takes a **closure** that returns an `App`.
Each route takes a **closure** that returns an **async block** (another closure-like
construct). Closures all the way down.

```
The Rust Abstraction Stack — Closures Are the Foundation
════════════════════════════════════════════════════════

  Web Frameworks (Axum, Actix, Rocket)
          │
          ▼
  Async Runtime (Tokio, async-std)
          │
          ▼
  Futures & async/await
          │
          ▼
  Iterators & Streams
          │
          ▼
  ┌────────────────────┐
  │     CLOSURES       │  ← you are here
  │  (Fn / FnMut /     │
  │   FnOnce traits)   │
  └────────────────────┘

  Everything above depends on closures.
  Master closures → unlock the entire stack.
════════════════════════════════════════════════════════
```

The connection is direct: **closures → iterators → async → web frameworks**.
Understanding closure capture semantics — how `Fn`, `FnMut`, and `FnOnce` work,
what `move` does, why each closure has a unique type — is not just academic.
It's the prerequisite for productive async Rust and web development. Every
concept you learn in this chapter pays dividends throughout the rest of the language.

---

## Exercises

### Exercise 1: Higher-Order Function

Write `apply_twice` that takes a closure and applies it twice:

```rust
fn apply_twice<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    todo!()
}
// apply_twice(|x| x + 3, 7) → 13
// apply_twice(|x| x * 2, 3) → 12
```

<details>
<summary>Solution</summary>

```rust
fn apply_twice<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(f(x))
}

fn main() {
    println!("{}", apply_twice(|x| x + 3, 7));   // 13
    println!("{}", apply_twice(|x| x * 2, 3));   // 12
}
```

</details>

### Exercise 2: Closure Factory

Write a function that returns a counter closure:

```rust
fn make_counter() -> impl FnMut() -> i32 {
    todo!()
}
// let mut c = make_counter();
// c() → 1, c() → 2, c() → 3
```

<details>
<summary>Solution</summary>

```rust
fn make_counter() -> impl FnMut() -> i32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}

fn main() {
    let mut c = make_counter();
    println!("{}", c());  // 1
    println!("{}", c());  // 2
    println!("{}", c());  // 3
}
```

</details>

---

## Summary

| Concept | Detail |
|---------|--------|
| Syntax | `\|params\| body` |
| Capture | Automatically by `&T`, `&mut T`, or value |
| `move` keyword | Forces capture by value |
| Type | Each closure has a unique anonymous type |
| As parameter | `impl Fn(...)` or generic `F: Fn(...)` |
| As return | `impl Fn(...)` or `Box<dyn Fn(...)>` |
| Vs function | Closures capture env; functions don't |

---

**Next:** [Closure Traits →](./02-closure-traits.md)

<p align="center"><i>Tutorial 1 of 7 — Stage 6: Closures and Iterators</i></p>

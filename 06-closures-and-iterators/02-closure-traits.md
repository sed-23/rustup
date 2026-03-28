# Closure Traits — Fn, FnMut, FnOnce 🎭

> **Every closure in Rust implements one or more of three traits: `Fn`, `FnMut`, or `FnOnce`. These traits describe HOW the closure uses its captured variables, and they determine where the closure can be used.**

---

## Table of Contents

- [The Three Closure Traits](#the-three-closure-traits)
- [FnOnce — Called at Most Once](#fnonce--called-at-most-once)
- [FnMut — Can Mutate Captures](#fnmut--can-mutate-captures)
- [Fn — Immutable Access Only](#fn--immutable-access-only)
- [Trait Hierarchy](#trait-hierarchy)
- [Choosing the Right Trait Bound](#choosing-the-right-trait-bound)
- [Real-World Examples](#real-world-examples)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Three Closure Traits

| Trait | Captures | Can call | Method signature |
|-------|----------|----------|-----------------|
| `FnOnce` | By value (consumes) | Once | `call_once(self)` |
| `FnMut` | By `&mut` reference | Many times | `call_mut(&mut self)` |
| `Fn` | By `&` reference | Many times | `call(&self)` |

Think of it as: what does the closure **do** with the things it captured?

```
Fn:     "I just look at my captures"       (read-only)
FnMut:  "I modify my captures"             (read-write)
FnOnce: "I consume/move out of my captures" (one-time use)
```

---

## FnOnce — Called at Most Once

A closure that **moves** a captured value out can only be called once:

```rust
fn main() {
    let name = String::from("Alice");
    
    // This closure MOVES name out (consumes it)
    let consume = || {
        let moved_name = name;  // Takes ownership
        println!("Consumed: {}", moved_name);
        // `name` is gone after this
    };
    
    consume();    // ✅ First (and only) call
    // consume(); // ❌ Can't call again — name was already moved
}
```

`FnOnce` is the **most permissive** bound — ALL closures implement it:

```rust
fn call_once<F: FnOnce()>(f: F) {
    f();  // Takes ownership of f via `self`
}

fn main() {
    let name = String::from("Alice");
    
    // Moving closure → FnOnce
    call_once(|| {
        drop(name);  // Consumes name
    });
    
    // Mutating closure → also FnOnce (superset)
    let mut x = 0;
    call_once(|| x += 1);
    
    // Pure closure → also FnOnce
    call_once(|| println!("hello"));
}
```

---

## FnMut — Can Mutate Captures

A closure that **modifies** captured variables (but doesn't consume them):

```rust
fn main() {
    let mut total = 0;
    
    let mut add_to_total = |x: i32| {
        total += x;  // Mutates captured `total`
    };
    
    add_to_total(10);
    add_to_total(20);
    add_to_total(30);
    
    println!("Total: {}", total);  // 60
}
```

Note: the closure variable itself must be `mut`:

```rust
fn call_twice<F: FnMut()>(mut f: F) {
    //                      ^^^ must be mut because we call f which takes &mut self
    f();
    f();
}

fn main() {
    let mut count = 0;
    call_twice(|| count += 1);
    println!("{}", count);  // 2
}
```

---

## Fn — Immutable Access Only

A closure that **only reads** captured variables (or captures nothing):

```rust
fn main() {
    let name = String::from("Alice");
    
    let greet = || {
        println!("Hello, {}!", name);  // Only reads `name`
    };
    
    greet();
    greet();
    greet();
    
    println!("{}", name);  // ✅ Still usable
}
```

`Fn` closures can be called from multiple places simultaneously:

```rust
fn call_many<F: Fn(i32) -> i32>(f: F) {
    println!("{}", f(1));
    println!("{}", f(2));
    println!("{}", f(3));
}

fn main() {
    let multiplier = 10;
    call_many(|x| x * multiplier);
    // 10
    // 20
    // 30
}
```

---

### Callable Types Across Programming Languages

Rust's three closure traits are unique in the programming language landscape. Most languages
have a **single** concept of "callable thing" — Rust distinguishes **how** the callable
interacts with its captured data at the type level. Let's compare:

#### C — Function Pointers Only

C has no closures at all. The only callable type is a **function pointer** (`void (*fn)(int)`).
Function pointers carry no captured state — if you need state, you pass it as an extra
argument (the classic `void *userdata` pattern). This is maximally efficient but ergonomically
painful.

#### C++ — Multiple Mechanisms

C++ has three callable mechanisms:
- **Function pointers** — same as C, no state
- **Function objects (functors)** — structs with `operator()` overloaded, can carry state
- **Lambdas** — compiler-generated classes with `operator()`, syntactic sugar for functors
- **`std::function<void(int)>`** — type-erased wrapper that can hold any callable (heap-allocates)

C++ lambdas have capture modes (`[x]` by value, `[&x]` by reference), but this isn't encoded
in the type system — a lambda that moves out of a capture can still be called twice, leading
to **undefined behavior**.

#### Java — Functional Interfaces

Java uses `@FunctionalInterface` — single-method interfaces like `Runnable`, `Function<T, R>`,
`Consumer<T>`. Lambdas are objects implementing these interfaces. They're always **heap-allocated**
and garbage-collected. There's no distinction between a lambda that reads vs. mutates vs.
consumes — it's all the same `Function<T, R>` type.

#### Python & JavaScript — Universal Callables

In Python, anything with a `__call__` method is callable. In JavaScript, all functions are
closures over their lexical scope. Neither language distinguishes closures from functions at
the type level. Both use garbage collection, so there's no concept of "consuming" a capture.

#### Haskell — Everything Is a Closure

In Haskell, every function is a closure (partially applied functions are the norm). Since
Haskell is purely functional and all data is immutable, there's no need to distinguish
between read/mutate/consume — it's always read-only.

#### Where Rust Stands

| Language     | Callable Types | Closure vs Function Distinction | Type-Level Capture Safety | Allocation |
|-------------|----------------|--------------------------------|--------------------------|------------|
| **C**       | Function pointers only | No closures exist | N/A | None (stack) |
| **C++**     | Fn ptrs, functors, lambdas, `std::function` | Capture modes exist but not in type | ❌ UB possible | Optional (stack or heap) |
| **Java**    | Functional interfaces | No — all are interface objects | ❌ All effectively final | Always heap |
| **Python**  | Any `__call__` | No distinction | ❌ No type system | Always heap (GC) |
| **JavaScript** | All functions | No distinction | ❌ No type system | Always heap (GC) |
| **Haskell** | All functions | No distinction (all immutable) | N/A (no mutation) | GC-managed |
| **Rust**    | `Fn`, `FnMut`, `FnOnce` | **Yes — encoded in the type system** | ✅ Compile-time enforced | Stack by default |

Rust is the **only mainstream language** where the type system encodes how a callable uses
its captured data. This enables the compiler to prevent use-after-move, data races on
closures, and accidental double-consumption — all at zero runtime cost.

---

## Trait Hierarchy

The three traits form a hierarchy:

```
FnOnce   ← ALL closures implement this (most general)
  ↑
FnMut    ← closures that don't consume captures
  ↑
Fn       ← closures that don't mutate captures (most restrictive)
```

Every `Fn` is also `FnMut`, and every `FnMut` is also `FnOnce`:

```
Fn ⊂ FnMut ⊂ FnOnce
```

```rust
fn takes_fn_once<F: FnOnce()>(f: F) { f(); }
fn takes_fn_mut<F: FnMut()>(mut f: F) { f(); }
fn takes_fn<F: Fn()>(f: F) { f(); }

fn main() {
    let s = String::from("hello");
    
    // Fn closure (only reads)
    let read = || println!("{}", s);
    takes_fn(&read);       // ✅ Fn
    takes_fn_mut(&read);   // ✅ Fn → FnMut
    takes_fn_once(&read);  // ✅ Fn → FnMut → FnOnce
    
    // FnMut closure (mutates)
    let mut x = 0;
    let mut mutate = || x += 1;
    // takes_fn(&mutate);     // ❌ Not Fn (it mutates)
    takes_fn_mut(&mut mutate);  // ✅ FnMut
    takes_fn_once(mutate);      // ✅ FnMut → FnOnce
    
    // FnOnce closure (consumes)
    let name = String::from("Alice");
    let consume = || drop(name);
    // takes_fn(&consume);     // ❌ Not Fn
    // takes_fn_mut(&mut c);   // ❌ Not FnMut
    takes_fn_once(consume);    // ✅ FnOnce only
}
```

---

### Why Three Traits? — The Ownership Connection

Rust's three closure traits aren't arbitrary — they directly mirror Rust's **three ways to access data**:

```
┌─────────────────────────────────────────────────────────────────┐
│   Access Mode       Closure Trait      Method Receiver         │
├─────────────────────────────────────────────────────────────────┤
│   &T   (shared ref)      Fn            call(&self)             │
│   &mut T (exclusive ref) FnMut         call_mut(&mut self)     │
│   T    (owned value)     FnOnce        call_once(self)         │
└─────────────────────────────────────────────────────────────────┘
```

This is the **same ownership model** applied everywhere else in Rust — closures are no exception:

- **`Fn`**: captures by **shared reference** (`&T`) — can be called unlimited times without changing anything. Like handing someone a book to read: they give it back exactly as it was.
- **`FnMut`**: captures by **mutable reference** (`&mut T`) — can be called repeatedly but modifies its state each time. Like handing someone a notebook to write in: they change it, but you still own it.
- **`FnOnce`**: captures **by value** (`T`) — consumes the captured data, can only be called once. Like giving someone a gift: once given, it's gone from your hands.

**No other mainstream language makes this distinction!**

| Language     | Closure Capture      | Memory Management | Trait Distinction? |
|--------------|----------------------|-------------------|--------------------|
| **Rust**     | `&T`, `&mut T`, `T`  | Ownership system  | ✅ Fn / FnMut / FnOnce |
| Java         | Effectively final ref | GC                | ❌ Single `Function` interface |
| Python       | Reference to scope   | GC                | ❌ Just callable |
| JavaScript   | Reference to scope   | GC                | ❌ Just callable |
| C++          | `[x]`, `[&x]`       | Manual / RAII     | ⚠️ Capture modes but no trait split |

C++'s capture lists (`[x]` by value, `[&x]` by reference) are the closest equivalent,
but C++ doesn't encode this in the **type system** — you can always call a C++ lambda
multiple times even if it moves out of its captures (leading to undefined behavior).
Rust **prevents this at compile time** via the trait hierarchy.

The hierarchy means you can always "widen" the bound:

```
Fn ⊂ FnMut ⊂ FnOnce

• Every Fn    closure is automatically FnMut  (reading is a subset of mutating)
• Every FnMut closure is automatically FnOnce (mutating is a subset of consuming)
```

So if your function accepts `impl FnOnce`, it accepts **ALL** closures — it's the most
permissive bound from the callee's perspective. Conversely, `Fn` is the most **useful**
bound from the caller's perspective (you can call it as many times as you want).

**Practical guideline:** When writing generic functions, start with the **most restrictive**
trait that works and relax only if needed:

```
1. Try Fn    first → maximum flexibility for callers (call many times, share across threads)
2. Try FnMut next  → needed if the closure must mutate state between calls
3. Use FnOnce last → when you only call the closure once (thread::spawn, Option::map, etc.)
```

---

### The Trait Hierarchy — Why Fn: FnMut: FnOnce

The relationship `Fn: FnMut: FnOnce` is a **subtrait** (supertrait) chain defined in the
standard library:

```rust
// Simplified from std:
pub trait FnOnce<Args> {
    type Output;
    fn call_once(self, args: Args) -> Self::Output;
}

pub trait FnMut<Args>: FnOnce<Args> {
    fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait Fn<Args>: FnMut<Args> {
    fn call(&self, args: Args) -> Self::Output;
}
```

This means: **every `Fn` is also an `FnMut`**, and **every `FnMut` is also an `FnOnce`**.
But why does this make sense?

#### The Logical Argument

- If a closure only **reads** its captures (`&self`), it can trivially satisfy a `&mut self`
  requirement — not mutating is a valid form of "mutating" (you just don't change anything).
- If a closure can be called **many times**, it can certainly be called **once** — calling
  once is a subset of calling many times.

This is the **Liskov Substitution Principle** in action: a more capable type can always
substitute for a less capable one without breaking correctness.

#### Real-World Analogy

Think of a chef's skill levels:
- **`Fn` chef**: Can make any dish perfectly, any number of times, without changing the kitchen.
- **`FnMut` chef**: Can make dishes, but rearranges the kitchen each time.
- **`FnOnce` chef**: Can make exactly one dish, then retires.

An `Fn` chef can clearly do everything an `FnMut` chef can (just choose not to rearrange).
An `FnMut` chef can do what an `FnOnce` chef does (just stop after one dish). The hierarchy
is natural.

#### Why This Matters in Practice

When writing a function that accepts a closure, use the **least restrictive** trait to
accept the **widest range** of closures:

```rust
// Accepts ALL closures: Fn, FnMut, AND FnOnce
fn execute_once<F: FnOnce() -> String>(f: F) -> String {
    f()
}

fn main() {
    // Fn closure — works!
    let greeting = "hello".to_string();
    let result = execute_once(|| greeting.clone());
    println!("{}", result);

    // FnMut closure — works!
    let mut count = 0;
    let result = execute_once(|| {
        count += 1;
        format!("count: {}", count)
    });
    println!("{}", result);

    // FnOnce closure — works!
    let name = String::from("Alice");
    let result = execute_once(|| {
        name  // moves name out — can only happen once
    });
    println!("{}", result);
}
```

If `execute_once` required `Fn` instead of `FnOnce`, the third closure would be **rejected**
by the compiler — even though we only call it once. By using `FnOnce`, we tell the compiler
"I only need to call this once," which maximizes the set of closures callers can pass in.

---

## Choosing the Right Trait Bound

**Rule: Use the most general trait your function needs:**

| If your function... | Use |
|---------------------|-----|
| Calls the closure once | `FnOnce` |
| Calls the closure multiple times, closure might mutate | `FnMut` |
| Calls the closure multiple times, no mutation | `Fn` |

```rust
// Only called once → FnOnce (most flexible for callers)
fn run_once<F: FnOnce() -> String>(f: F) -> String {
    f()
}

// Called in a loop → FnMut
fn repeat<F: FnMut()>(mut f: F, times: usize) {
    for _ in 0..times {
        f();
    }
}

// Shared across threads, no mutation → Fn
fn parallel_apply<F: Fn(i32) -> i32 + Send + Sync>(f: &F, data: &[i32]) -> Vec<i32> {
    data.iter().map(|&x| f(x)).collect()
}
```

Standard library examples:

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    // map takes FnMut (general enough, called multiple times)
    let doubled: Vec<i32> = v.iter().map(|&x| x * 2).collect();
    
    // unwrap_or_else takes FnOnce (called at most once)
    let val: i32 = None.unwrap_or_else(|| 42);
    
    // sort_by takes FnMut (called multiple times during sorting)
    let mut nums = vec![3, 1, 2];
    nums.sort_by(|a, b| a.cmp(b));
}
```

---

## Real-World Examples

### Callbacks

```rust
struct Button {
    on_click: Box<dyn FnMut()>,
}

impl Button {
    fn new(on_click: impl FnMut() + 'static) -> Self {
        Button {
            on_click: Box::new(on_click),
        }
    }
    
    fn click(&mut self) {
        (self.on_click)();
    }
}

fn main() {
    let mut click_count = 0;
    
    // Need to use move + interior mutability for shared state
    // For simplicity, just demonstrate the pattern:
    let mut btn = Button::new(|| {
        println!("Button clicked!");
    });
    
    btn.click();
    btn.click();
}
```

### Lazy Evaluation

```rust
struct Lazy<T, F: FnOnce() -> T> {
    init: Option<F>,
    value: Option<T>,
}

impl<T, F: FnOnce() -> T> Lazy<T, F> {
    fn new(f: F) -> Self {
        Lazy { init: Some(f), value: None }
    }
    
    fn get(&mut self) -> &T {
        if self.value.is_none() {
            let f = self.init.take().unwrap();
            self.value = Some(f());  // FnOnce: called exactly once
        }
        self.value.as_ref().unwrap()
    }
}

fn main() {
    let mut expensive = Lazy::new(|| {
        println!("Computing...");
        42
    });
    
    println!("{}", expensive.get());  // "Computing..." then "42"
    println!("{}", expensive.get());  // Just "42" (cached)
}
```

### Strategy Pattern

```rust
fn process_numbers(
    nums: &[i32],
    filter: impl Fn(&i32) -> bool,
    transform: impl Fn(i32) -> i32,
) -> Vec<i32> {
    nums.iter()
        .filter(|&&x| filter(&x))
        .map(|&x| transform(x))
        .collect()
}

fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    let result = process_numbers(
        &nums,
        |&x| x % 2 == 0,    // Filter: even numbers
        |x| x * x,           // Transform: square
    );
    
    println!("{:?}", result);  // [4, 16, 36, 64, 100]
}
```

---

### Closure Traits and the Standard Library

The standard library uses closure traits **extensively**. Understanding which trait each
function requires — and **why** — will deepen your intuition for the whole system.

#### Key Standard Library Signatures

```rust
// Iterator methods (called potentially many times per element)
fn map<B, F>(self, f: F) -> Map<Self, F>
    where F: FnMut(Self::Item) -> B;          // FnMut — may track state across calls

fn filter<P>(self, predicate: P) -> Filter<Self, P>
    where P: FnMut(&Self::Item) -> bool;      // FnMut — same reason

fn for_each<F>(self, f: F)
    where F: FnMut(Self::Item);               // FnMut — side effects expected

// Option / Result methods (called at most once)
fn map<U, F>(self, f: F) -> Option<U>
    where F: FnOnce(T) -> U;                  // FnOnce — only one value in Some

fn unwrap_or_else<F>(self, f: F) -> T
    where F: FnOnce() -> T;                   // FnOnce — called once on None

// Threading (runs once, takes ownership)
fn spawn<F, T>(f: F) -> JoinHandle<T>
    where F: FnOnce() -> T + Send + 'static;  // FnOnce — thread runs closure once
```

Notice the pattern:

| Function            | Trait      | Why This Trait?                                     |
|---------------------|------------|-----------------------------------------------------|
| `Iterator::map`     | `FnMut`    | Called per element; closure may track a running index |
| `Iterator::filter`  | `FnMut`    | Called per element; might count filtered items        |
| `Iterator::for_each`| `FnMut`    | Explicitly for side effects (mutation expected)       |
| `Option::map`       | `FnOnce`   | At most one `Some` value — called 0 or 1 times       |
| `Option::unwrap_or_else` | `FnOnce` | Called once when `None`                          |
| `Result::map_err`   | `FnOnce`   | At most one `Err` — called 0 or 1 times              |
| `thread::spawn`     | `FnOnce`   | Thread takes ownership, runs body exactly once        |
| `Vec::sort_by`      | `FnMut`    | Comparator called O(n log n) times during sorting     |
| `Vec::retain`       | `FnMut`    | Predicate called once per element                     |

**Key insight:** Very few standard library functions require `Fn` specifically. `FnMut` is
almost always sufficient because it also accepts `Fn` closures (remember the hierarchy).
The stdlib chooses `FnMut` over `Fn` to give **callers maximum flexibility** — a stateful
closure like a counter works just fine:

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Charlie"];

    // map uses FnMut, so we CAN track state:
    let mut index = 0;
    let numbered: Vec<String> = names.iter().map(|name| {
        index += 1;  // mutation! — requires FnMut, not Fn
        format!("{}. {}", index, name)
    }).collect();

    println!("{:?}", numbered);
    // ["1. Alice", "2. Bob", "3. Charlie"]
}
```

If `map` required `Fn` instead of `FnMut`, the closure above would be rejected — the
stdlib deliberately uses the **least restrictive trait** that's safe.

---

## Exercises

### Exercise 1: Which Trait?

For each closure, determine which trait(s) it implements:

```rust
let a = || println!("hello");          // ?
let mut x = 0;
let b = || { x += 1; };               // ?
let s = String::from("bye");
let c = || { drop(s); };              // ?
let d = |x: i32, y: i32| x + y;      // ?
```

<details>
<summary>Answer</summary>

```
a: Fn + FnMut + FnOnce (no captures, just prints)
b: FnMut + FnOnce (mutates x)
c: FnOnce only (consumes s via drop)
d: Fn + FnMut + FnOnce (no captures, pure computation)
```

</details>

### Exercise 2: Generic Retry

Write a function that retries an operation up to `n` times:

```rust
fn retry<F: FnMut() -> Result<T, E>, T, E>(mut f: F, attempts: usize) -> Result<T, E> {
    todo!()
}
```

<details>
<summary>Solution</summary>

```rust
fn retry<F, T, E>(mut f: F, attempts: usize) -> Result<T, E>
where
    F: FnMut() -> Result<T, E>,
{
    let mut last_err = None;
    for _ in 0..attempts {
        match f() {
            Ok(val) => return Ok(val),
            Err(e) => last_err = Some(e),
        }
    }
    Err(last_err.unwrap())
}

fn main() {
    let mut attempt = 0;
    let result = retry(|| {
        attempt += 1;
        if attempt < 3 {
            Err(format!("Failed on attempt {}", attempt))
        } else {
            Ok("Success!")
        }
    }, 5);
    
    println!("{:?}", result);  // Ok("Success!")
}
```

</details>

---

### How Closure Traits Compile — The Desugaring

Ever wonder what the compiler actually **does** with a closure? Each closure is transformed
into an **anonymous struct** with a trait implementation. Understanding this desugaring
demystifies closure behavior.

#### A Simple `Fn` Closure

```rust
// What you write:
let captured = 10;
let add = |x: i32| x + captured;
println!("{}", add(5));  // 15
```

The compiler transforms this into something equivalent to:

```rust
// What the compiler generates (approximately):
struct __ClosureAdd {
    captured: i32,      // stored by shared reference internally
}

impl Fn(i32,) -> i32 for __ClosureAdd {
    fn call(&self, (x,): (i32,)) -> i32 {
        x + self.captured   // &self — only reads
    }
}

let add = __ClosureAdd { captured: 10 };
println!("{}", add.call((5,)));  // 15
```

#### An `FnMut` Closure

If the closure mutates captured state, the method takes `&mut self`:

```rust
// What you write:
let mut count = 0;
let mut inc = || { count += 1; };

// Desugars roughly to:
struct __ClosureInc<'a> {
    count: &'a mut i32,
}
impl FnMut() for __ClosureInc<'_> {
    fn call_mut(&mut self) {
        *self.count += 1;   // &mut self — mutates
    }
}
```

#### An `FnOnce` Closure

If the closure moves out of a capture, the method takes `self` (by value):

```rust
// What you write:
let name = String::from("Alice");
let consume = || { drop(name); };

// Desugars roughly to:
struct __ClosureConsume {
    name: String,           // owned — moved into the struct
}
impl FnOnce() for __ClosureConsume {
    fn call_once(self) {    // self by value — struct is consumed
        drop(self.name);
    }
}
```

#### Why This Matters

This desugaring explains several things that might otherwise seem mysterious:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Observation                    │  Explained By                    │
├─────────────────────────────────┼──────────────────────────────────┤
│  Every closure has a unique     │  Each anonymous struct is a      │
│  type (can't name it)           │  distinct compiler-generated type│
├─────────────────────────────────┼──────────────────────────────────┤
│  Closures are zero-cost         │  The struct + call can be fully  │
│  abstractions                   │  inlined by LLVM — no allocation │
├─────────────────────────────────┼──────────────────────────────────┤
│  FnOnce can only be called once │  call_once(self) consumes the    │
│                                 │  struct — no struct left to call │
├─────────────────────────────────┼──────────────────────────────────┤
│  You need `dyn Fn` or Box for   │  Without type erasure, each      │
│  heterogeneous closure storage  │  closure is a different type     │
└─────────────────────────────────┴──────────────────────────────────┘
```

In optimized builds, the anonymous struct often **doesn't even exist** — LLVM inlines the
captured fields and the call body directly into the surrounding function. This is why Rust
closures can match hand-written loops in performance.

**Comparison with other languages:**
- **Java** lambdas create an object with a `call`-like method — conceptually similar, but
  allocated on the heap and managed by the garbage collector.
- **C++** lambdas also generate anonymous types with `operator()`, and can also be inlined.
  However, C++ lacks the trait-level distinction, so the compiler can't enforce call-once
  semantics at the type level.
- **Python/JavaScript** closures hold references to scope variables — no struct, no inlining,
  always dynamically dispatched.

The trade-off: Rust's closures are **faster** (no allocation, no GC, no dynamic dispatch by
default) but require **more type system complexity** (three traits, unique anonymous types,
lifetime tracking). For systems programming, this trade-off is almost always worth it.

---

### Dynamic Dispatch with `dyn Fn` — When Generics Aren't Enough

So far we've used closures with **static dispatch** — the compiler knows the exact closure
type at compile time and monomorphizes each call site. This is zero-cost but has a limitation:
each closure has a **unique anonymous type**, so you can't mix different closures in the
same container.

#### The Problem

```rust
fn main() {
    let add_one = |x: i32| x + 1;
    let double  = |x: i32| x * 2;

    // ❌ Won't compile — each closure is a DIFFERENT type!
    // let transforms: Vec<???> = vec![add_one, double];
}
```

Even though both closures have the signature `i32 -> i32`, they are distinct anonymous types.
Generics and `impl Fn` can't help here because they resolve to **one** concrete type.

#### The Solution: `dyn Fn` with `Box`

```rust
fn main() {
    let add_one: Box<dyn Fn(i32) -> i32> = Box::new(|x| x + 1);
    let double:  Box<dyn Fn(i32) -> i32> = Box::new(|x| x * 2);
    let negate:  Box<dyn Fn(i32) -> i32> = Box::new(|x| -x);

    // ✅ Different closures, same container!
    let transforms: Vec<Box<dyn Fn(i32) -> i32>> = vec![add_one, double, negate];

    let input = 5;
    for (i, f) in transforms.iter().enumerate() {
        println!("transform[{}]({}) = {}", i, input, f(input));
    }
    // transform[0](5) = 6
    // transform[1](5) = 10
    // transform[2](5) = -5
}
```

#### How the Vtable Works

A `Box<dyn Fn(i32) -> i32>` is a **fat pointer** consisting of two machine words:

```
┌──────────────────────────────────────────────────────────┐
│  Box<dyn Fn(i32) -> i32>                                 │
├────────────────────────┬─────────────────────────────────┤
│  ptr to closure data   │  ptr to vtable                  │
│  (captured variables)  │  ┌───────────────────────────┐  │
│                        │  │ call(&self, i32) -> i32   │  │
│                        │  │ drop(self)                │  │
│                        │  │ size                      │  │
│                        │  │ alignment                 │  │
│                        │  └───────────────────────────┘  │
└────────────────────────┴─────────────────────────────────┘
```

Each call goes through **one pointer indirection** to look up the `call` method in the
vtable. This costs roughly 1–2 nanoseconds extra — negligible in most applications, but
measurable in tight inner loops processing millions of elements.

#### When to Use `dyn Fn`

| Scenario | Use `impl Fn` / Generics | Use `Box<dyn Fn>` |
|----------|--------------------------|--------------------|
| Single closure, known at compile time | ✅ Zero-cost | Unnecessary overhead |
| Storing multiple closures in a `Vec` | ❌ Can't — different types | ✅ Type-erased |
| Returning different closures from `match` | ❌ Each arm has different type | ✅ Unified return type |
| Callback registries / event handlers | ❌ One generic = one type | ✅ Multiple handlers |
| Plugin systems / middleware chains | ❌ Types unknown at compile time | ✅ Dynamic composition |

#### Real-World Pattern: Event Handler Registry

```rust
struct EventBus {
    handlers: Vec<Box<dyn Fn(&str)>>,
}

impl EventBus {
    fn new() -> Self {
        EventBus { handlers: Vec::new() }
    }

    fn on_event(&mut self, handler: impl Fn(&str) + 'static) {
        self.handlers.push(Box::new(handler));
    }

    fn emit(&self, event: &str) {
        for handler in &self.handlers {
            handler(event);  // dynamic dispatch through vtable
        }
    }
}

fn main() {
    let mut bus = EventBus::new();

    bus.on_event(|e| println!("Logger: {}", e));
    bus.on_event(|e| println!("Metrics: recording {}", e));
    bus.on_event(|e| {
        if e.starts_with("error") {
            println!("Alert: {}", e);
        }
    });

    bus.emit("user.login");
    bus.emit("error.timeout");
}
```

#### Contrast with Other Languages

In Java and JavaScript, **all** callables are dynamically dispatched — there's no choice.
You always pay the indirection cost, even when the compiler could theoretically inline.
Rust gives you the **choice**: use generics for the hot path (zero-cost), and `dyn Fn` for
flexibility when you need heterogeneous collections. This "pay only for what you use"
philosophy is a core Rust principle.

---

## Summary

| Trait | Method | Self | Can Call | Use When |
|-------|--------|------|----------|----------|
| `FnOnce` | `call_once` | `self` | Once | Closure might consume captures |
| `FnMut` | `call_mut` | `&mut self` | Many | Closure might mutate captures |
| `Fn` | `call` | `&self` | Many | Closure only reads captures |

**Hierarchy:** `Fn` ⊂ `FnMut` ⊂ `FnOnce`

**Rule of thumb:** Use `FnOnce` unless you need to call multiple times, then use `FnMut`, and only require `Fn` if you need shared/immutable access.

---

**Previous:** [← Closures](./01-closures.md) · **Next:** [Closures and Ownership →](./03-closures-and-ownership.md)

<p align="center"><i>Tutorial 2 of 7 — Stage 6: Closures and Iterators</i></p>

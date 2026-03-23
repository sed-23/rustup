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

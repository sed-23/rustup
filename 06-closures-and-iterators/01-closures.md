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

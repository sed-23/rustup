# Generic Functions 🔀

> **"Write code that works for many types, not just one."**

Generics let you write a single function that operates on values of *any* type — while still catching mistakes at compile time. This chapter starts from the duplication problem and builds up to Rust's zero-cost generic model.

---

## Table of Contents

1. [The Duplication Problem](#1-the-duplication-problem)
2. [Generic Syntax](#2-generic-syntax)
3. [The Constraint Problem — Trait Bounds](#3-the-constraint-problem--trait-bounds)
4. [Monomorphization — Zero-Cost Generics](#4-monomorphization--zero-cost-generics)
5. [Generics Across Languages](#5-generics-across-languages)
6. [A Brief History of Generics](#6-a-brief-history-of-generics)
7. [Common Generic Patterns](#7-common-generic-patterns)
8. [Exercises](#8-exercises)
9. [Summary](#9-summary)

---

## 1. The Duplication Problem

Imagine you need a function that returns the largest element in a slice. You start with integers:

```rust
fn largest_i32(list: &[i32]) -> &i32 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

Then you need the same logic for `f64`:

```rust
fn largest_f64(list: &[f64]) -> &f64 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

And again for `char`:

```rust
fn largest_char(list: &[char]) -> &char {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

Calling them:

```rust
fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    println!("Largest i32: {}", largest_i32(&numbers));

    let floats = vec![1.5, 3.7, 2.2, 9.1];
    println!("Largest f64: {}", largest_f64(&floats));

    let chars = vec!['y', 'm', 'a', 'q'];
    println!("Largest char: {}", largest_char(&chars));
}
```

All three function bodies are **identical**. The only difference is the type signature. This is a textbook case for generics — the logic is the same, only the types change.

### Why Copy-Paste Hurts

| Problem            | Impact                                          |
| ------------------ | ----------------------------------------------- |
| Bug in one copy    | Must fix all copies — easy to miss one          |
| New type needed    | Write yet another copy                          |
| Readability        | Readers must verify bodies are truly identical   |
| Maintenance burden | Every change multiplied by the number of copies  |

---

## 2. Generic Syntax

### The Basic Form

Replace the concrete type with a **type parameter** enclosed in angle brackets:

```rust
fn largest<T>(list: &[T]) -> &T {
    // ...
}
```

- `<T>` declares a type parameter named `T`.
- `T` is a placeholder — the caller fills it in with a real type.
- By convention, single uppercase letters are used: `T`, `U`, `V`, etc.

### Naming Conventions

| Parameter | Common Use                        |
| --------- | --------------------------------- |
| `T`       | General "type"                    |
| `U`, `V`  | Second and third type parameters  |
| `K`, `V`  | Key and value (maps)              |
| `E`       | Error type                        |
| `R`       | Return type                       |
| `S`       | State                             |

You can also use descriptive names when clarity helps:

```rust
fn convert<Input, Output>(value: Input) -> Output {
    // ...
}
```

### Multiple Type Parameters

A function can accept more than one generic type:

```rust
fn pair<T, U>(first: T, second: U) -> (T, U) {
    (first, second)
}

fn main() {
    let p = pair(42, "hello");
    println!("{:?}", p); // (42, "hello")
}
```

Each type parameter is independent — `T` and `U` may be the same concrete type or different types.

### Inference

Rust infers generic types from context, so you rarely need the "turbofish" syntax:

```rust
// Inferred — compiler sees i32 and &str from the arguments
let p = pair(42, "hello");

// Explicit turbofish — sometimes necessary
let p = pair::<i32, &str>(42, "hello");
```

The turbofish (`::<>`) is mainly needed when the compiler cannot infer the type, such as when parsing a string:

```rust
let num = "42".parse::<i32>().unwrap();
```

---

## 3. The Constraint Problem — Trait Bounds

Let's try to make `largest` generic:

```rust
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {   // ❌ ERROR
            largest = item;
        }
    }
    largest
}
```

The compiler rejects this:

```
error[E0369]: binary operation `>` cannot be applied to type `&T`
  |
  |         if item > largest {
  |            ---- ^ ------- &T
  |            |
  |            &T
  |
help: consider restricting type parameter `T`
  |
  | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
```

**Why?** The compiler doesn't know that `T` supports `>`. Not every type can be compared — think of a struct with no ordering. We need a **trait bound** to promise that `T` implements `PartialOrd`.

### Adding the Bound

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

Now it compiles. The `T: PartialOrd` bound says: "T must implement the `PartialOrd` trait." We'll explore trait bounds in depth in the next few tutorials — for now, just know that bounds restrict which types can fill a generic parameter.

### A Complete Working Example

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    println!("Largest number: {}", largest(&numbers));

    let chars = vec!['y', 'm', 'a', 'q'];
    println!("Largest char: {}", largest(&chars));

    let floats = vec![1.5, 3.7, 2.2, 9.1];
    println!("Largest float: {}", largest(&floats));
}
```

Output:

```
Largest number: 100
Largest char: y
Largest float: 9.1
```

One function, three types — zero duplication.

---

## 4. Monomorphization — Zero-Cost Generics

### What Happens at Compile Time

When you write a generic function and call it with `i32` and `char`, the compiler generates **two separate, specialized functions** — one for each concrete type. This process is called **monomorphization**.

```
  Source Code                  After Monomorphization
 ─────────────                ───────────────────────

 fn largest<T: PartialOrd>    fn largest_i32(list: &[i32]) -> &i32 { ... }
   (list: &[T]) -> &T         fn largest_char(list: &[char]) -> &char { ... }
   { ... }                    fn largest_f64(list: &[f64]) -> &f64 { ... }
```

### ASCII Diagram — The Monomorphization Pipeline

```
┌──────────────────────────────────────────────────────────────────┐
│                        SOURCE CODE                               │
│                                                                  │
│   fn largest<T: PartialOrd>(list: &[T]) -> &T { ... }           │
│                                                                  │
│   largest(&[34, 50, 25]);       // call site 1: T = i32         │
│   largest(&['y', 'm', 'a']);    // call site 2: T = char        │
│   largest(&[1.5, 3.7]);         // call site 3: T = f64         │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                     MONOMORPHIZATION                             │
│                                                                  │
│   The compiler inspects every call site, discovers which         │
│   concrete types are used, and stamps out a specialized          │
│   copy for each one.                                             │
└──────────────────────┬───────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ largest_i32│ │largest_char│ │ largest_f64│
   │  (native)  │ │  (native)  │ │  (native)  │
   └────────────┘ └────────────┘ └────────────┘
          │            │            │
          └────────────┼────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                      FINAL BINARY                                │
│                                                                  │
│   Contains three specialized machine-code functions.             │
│   No runtime dispatch. No type metadata. No overhead.            │
└──────────────────────────────────────────────────────────────────┘
```

### Zero-Cost Abstraction

Because each specialization is compiled to native machine code with full optimizations:

- **No runtime dispatch** — the CPU jumps directly to the right code.
- **No boxing or heap allocation** — values stay on the stack.
- **Full inlining** — the optimizer can inline the specialized version.
- **Identical performance** — a generic function is exactly as fast as one you wrote by hand for that specific type.

This is what Rust means by **zero-cost abstractions**: you pay nothing at runtime for using generics.

### The Binary Size Tradeoff

There's a cost, but it's at compile time, not runtime:

| Aspect          | Generics (Monomorphization)        | Dynamic Dispatch (dyn Trait) |
| --------------- | ---------------------------------- | ---------------------------- |
| Runtime speed   | Fastest — fully specialized        | Slight overhead (vtable)     |
| Binary size     | Larger — one copy per type used    | Smaller — one shared copy    |
| Compile time    | Slower — more code to generate     | Faster                       |
| Flexibility     | Types known at compile time        | Types can vary at runtime    |

If you call `largest` with 10 different types, you get 10 copies of the function in your binary. For most programs this is fine, but in embedded or size-sensitive contexts, dynamic dispatch (`dyn Trait`) may be preferred. We'll cover that later in this stage.

### Verifying Monomorphization

You can inspect the generated symbols to see monomorphization in action:

```bash
# Build in release mode
cargo build --release

# List symbols (Linux/macOS)
nm target/release/my_app | grep largest

# On Windows with MSVC
dumpbin /symbols target\release\my_app.exe | findstr largest
```

You'll see separate mangled symbols for each concrete instantiation.

---

## 5. Generics Across Languages

Generics aren't unique to Rust. Different languages tackle the same problem in very different ways.

### C — `void*` (Unsafe Erasure)

C has no generics. The common workaround is `void*` pointers:

```c
void swap(void *a, void *b, size_t size) {
    char temp[size];
    memcpy(temp, a, size);
    memcpy(a, b, size);
    memcpy(b, temp, size);
}
```

- No type safety — the compiler can't check that `a` and `b` are the same type.
- Manual size tracking — pass `size` everywhere.
- Easy to cause undefined behavior.

### C++ — Templates

C++ templates are syntactically similar to Rust generics:

```cpp
template <typename T>
T largest(const std::vector<T>& list) {
    T result = list[0];
    for (const auto& item : list) {
        if (item > result) result = item;
    }
    return result;
}
```

- Also monomorphized — each instantiation generates code.
- Error messages can be notoriously cryptic (improved with C++20 Concepts).
- No trait bounds until C++20 Concepts — any operation is attempted and fails at instantiation.

### Java — Type Erasure

```java
public static <T extends Comparable<T>> T largest(List<T> list) {
    T result = list.get(0);
    for (T item : list) {
        if (item.compareTo(result) > 0) result = item;
    }
    return result;
}
```

- **Type erasure** — generic type info is removed at compile time.
- At runtime, `List<String>` and `List<Integer>` are both just `List`.
- Cannot use primitives (`int`, `double`) — must use boxed types (`Integer`, `Double`).
- No monomorphization — one shared copy with casts.

### C# — Reified Generics

```csharp
public static T Largest<T>(List<T> list) where T : IComparable<T> {
    T result = list[0];
    foreach (T item in list) {
        if (item.CompareTo(result) > 0) result = item;
    }
    return result;
}
```

- **Reified** — type info is preserved at runtime.
- Works with value types (no boxing for `int`, `float`).
- JIT specializes for value types, shares code for reference types.

### Go — Generics (since 1.18)

```go
func Largest[T constraints.Ordered](list []T) T {
    result := list[0]
    for _, item := range list {
        if item > result {
            result = item
        }
    }
    return result
}
```

- Added in Go 1.18 (March 2022).
- Uses **GC shape stenciling** — a hybrid approach.
- Shares code for types with the same GC shape, uses dictionaries for specifics.

### Comparison Table

| Feature             | C (`void*`) | C++ (Templates) | Java (Erasure) | C# (Reified) | Go (1.18)  | **Rust**       |
| ------------------- | ----------- | --------------- | -------------- | ------------- | ---------- | -------------- |
| Type safety         | ❌ None     | ⚠️ At instantiation | ✅ At compile time | ✅ At compile time | ✅ At compile time | ✅ At compile time |
| Runtime overhead    | None        | None            | Boxing/casts   | JIT-optimized | Dict lookup | **None**       |
| Primitives allowed  | N/A         | ✅              | ❌ (boxed)     | ✅            | ✅         | ✅             |
| Constraint system   | None        | Concepts (C++20)| Bounded types  | `where` clause| Constraints| **Trait bounds** |
| Code generation     | Manual      | Monomorphized   | Shared (erased)| Hybrid        | Hybrid     | **Monomorphized** |
| Error clarity       | N/A         | ⚠️ Verbose      | ✅ Clear       | ✅ Clear      | ✅ Clear   | ✅ Clear       |

---

## 6. A Brief History of Generics

| Year | Milestone                                                                 |
| ---- | ------------------------------------------------------------------------- |
| 1973 | **ML** introduces parametric polymorphism — the first generic type system |
| 1983 | **Ada** adds generic packages and procedures                              |
| 1985 | **C++** introduces templates (standardized in 1990)                       |
| 1990 | **Haskell** launches with type classes — inspiration for Rust traits      |
| 1999 | **Java** community begins JSR 14 process for generics                     |
| 2004 | **Java 5** ships generics via type erasure                                |
| 2005 | **C# 2.0** ships reified generics                                         |
| 2012 | **Rust 0.4** has generics from early on; trait bounds evolve over time    |
| 2015 | **Rust 1.0** stabilizes with the generics + trait system we know today    |
| 2020 | **C++20** adds Concepts — explicit constraints on templates               |
| 2022 | **Go 1.18** finally adds generics after years of community demand         |

The core idea — **write once, work for many types** — has been evolving for over 50 years, with each language refining the tradeoffs between safety, performance, and ergonomics.

---

## 7. Common Generic Patterns

### Identity Function

Returns its argument unchanged — the simplest possible generic function:

```rust
fn identity<T>(value: T) -> T {
    value
}

fn main() {
    let x = identity(42);       // T = i32
    let s = identity("hello");  // T = &str
    println!("{x}, {s}");
}
```

### Swap

Exchange two values of the same type:

```rust
fn swap<T>(a: &mut T, b: &mut T) {
    std::mem::swap(a, b);
}

fn main() {
    let mut x = 1;
    let mut y = 2;
    swap(&mut x, &mut y);
    println!("x={x}, y={y}"); // x=2, y=1
}
```

### Conversion / Transform

Apply a function to convert from one type to another:

```rust
fn transform<T, U, F>(value: T, func: F) -> U
where
    F: Fn(T) -> U,
{
    func(value)
}

fn main() {
    let length = transform("hello", |s: &str| s.len());
    println!("Length: {length}"); // Length: 5

    let doubled = transform(21, |n: i32| n * 2);
    println!("Doubled: {doubled}"); // Doubled: 42
}
```

### `where` Clause vs Inline Bounds

Both forms are equivalent. Use `where` when bounds get long:

**Inline bounds:**

```rust
fn print_largest<T: PartialOrd + std::fmt::Display>(list: &[T]) {
    let result = largest(list);
    println!("The largest is {result}");
}
```

**`where` clause:**

```rust
fn print_largest<T>(list: &[T])
where
    T: PartialOrd + std::fmt::Display,
{
    let result = largest(list);
    println!("The largest is {result}");
}
```

When multiple type parameters each have multiple bounds, `where` is much clearer:

```rust
fn complex<T, U, V>(t: T, u: U, v: V) -> String
where
    T: Display + Clone,
    U: Debug + PartialEq,
    V: Into<String>,
{
    format!("{t} | {u:?} | {}", v.into())
}
```

### Returning Generic Types

A function can return a generic type, though the caller must be able to determine what that type is:

```rust
fn first_or_default<T: Default>(list: &[T]) -> T
where
    T: Clone,
{
    match list.first() {
        Some(val) => val.clone(),
        None => T::default(),
    }
}

fn main() {
    let nums: Vec<i32> = vec![10, 20, 30];
    println!("{}", first_or_default(&nums)); // 10

    let empty: Vec<i32> = vec![];
    println!("{}", first_or_default(&empty)); // 0 (i32::default())
}
```

### Generic Helper: `min` and `max`

```rust
fn min_of<T: PartialOrd>(a: T, b: T) -> T {
    if a < b { a } else { b }
}

fn max_of<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

fn main() {
    println!("{}", min_of(3, 7));       // 3
    println!("{}", max_of(3.14, 2.72)); // 3.14
    println!("{}", min_of('a', 'z'));    // a
}
```

---

## 8. Exercises

### Exercise 1 — Generic `second_largest`

Write a function `second_largest` that returns a reference to the second largest element in a slice. Assume the slice has at least two elements.

**Signature hint:**

```rust
fn second_largest<T: PartialOrd>(list: &[T]) -> &T
```

<details>
<summary><strong>Solution</strong></summary>

```rust
fn second_largest<T: PartialOrd>(list: &[T]) -> &T {
    assert!(list.len() >= 2, "Need at least two elements");

    let mut first = &list[0];
    let mut second = &list[1];

    if second > first {
        std::mem::swap(&mut first, &mut second);
    }

    for item in &list[2..] {
        if item > first {
            second = first;
            first = item;
        } else if item > second {
            second = item;
        }
    }

    second
}

fn main() {
    let nums = vec![10, 40, 30, 20, 50];
    println!("Second largest: {}", second_largest(&nums)); // 40

    let chars = vec!['d', 'a', 'c', 'b'];
    println!("Second largest: {}", second_largest(&chars)); // c
}
```

</details>

---

### Exercise 2 — Generic `contains`

Write a function `contains` that checks whether a slice contains a given value. Return `true` if found, `false` otherwise.

**Signature hint:**

```rust
fn contains<T: PartialEq>(list: &[T], target: &T) -> bool
```

<details>
<summary><strong>Solution</strong></summary>

```rust
fn contains<T: PartialEq>(list: &[T], target: &T) -> bool {
    for item in list {
        if item == target {
            return true;
        }
    }
    false
}

fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    println!("{}", contains(&numbers, &3)); // true
    println!("{}", contains(&numbers, &9)); // false

    let words = vec!["apple", "banana", "cherry"];
    println!("{}", contains(&words, &"banana")); // true
    println!("{}", contains(&words, &"grape"));  // false
}
```

**Note:** The standard library provides `slice.contains()`, which does the same thing. This exercise is about practicing the generic syntax.

</details>

---

### Exercise 3 — Generic `zip_with`

Write a function `zip_with` that takes two slices and a closure, applies the closure to each pair of elements, and returns a `Vec` of results.

**Signature hint:**

```rust
fn zip_with<T, U, V, F>(a: &[T], b: &[U], func: F) -> Vec<V>
where
    F: Fn(&T, &U) -> V,
```

<details>
<summary><strong>Solution</strong></summary>

```rust
fn zip_with<T, U, V, F>(a: &[T], b: &[U], func: F) -> Vec<V>
where
    F: Fn(&T, &U) -> V,
{
    let len = a.len().min(b.len());
    let mut result = Vec::with_capacity(len);
    for i in 0..len {
        result.push(func(&a[i], &b[i]));
    }
    result
}

fn main() {
    let nums = vec![1, 2, 3];
    let strs = vec!["one", "two", "three"];

    let combined = zip_with(&nums, &strs, |n, s| format!("{n}: {s}"));
    println!("{:?}", combined);
    // ["1: one", "2: two", "3: three"]

    let a = vec![10, 20, 30];
    let b = vec![1, 2, 3];
    let sums = zip_with(&a, &b, |x, y| x + y);
    println!("{:?}", sums);
    // [11, 22, 33]
}
```

</details>

---

## 9. Summary

| Concept               | Key Idea                                                        |
| --------------------- | --------------------------------------------------------------- |
| Duplication problem   | Identical logic repeated for each type → use generics           |
| Type parameter `<T>`  | Placeholder filled by a concrete type at each call site         |
| Multiple params       | `<T, U>` — each independently resolved                         |
| Trait bounds          | `T: PartialOrd` restricts `T` to types that support comparison |
| `where` clause        | Cleaner syntax for complex bounds                               |
| Monomorphization      | Compiler stamps out specialized code per type — zero runtime cost |
| Binary size tradeoff  | More instantiations → more code in the binary                  |
| Turbofish `::<>`      | Explicit type annotation when inference isn't enough            |
| Common patterns       | identity, swap, transform, min/max                              |

### Key Takeaway

Rust generics give you **the flexibility of writing code once** and **the performance of hand-specialized code**. The compiler does the duplication for you — correctly, every time.

---

<p align="center">
  <strong>Tutorial 1 of 9 — Stage 8: Generics & Traits</strong>
</p>

<p align="center">
  Next: <a href="./02-generic-structs-enums.md">Generic Structs & Enums →</a>
</p>

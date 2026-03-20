# References & Borrowing 🔗

> **Borrowing lets you USE a value without taking ownership. It solves the tedium of moving values around. A reference is like looking at someone's book without taking it home.**

---

## Table of Contents

- [The Problem That Borrowing Solves](#the-problem-that-borrowing-solves)
- [What Is a Reference?](#what-is-a-reference)
  - [Creating References](#creating-references)
  - [References in Memory](#references-in-memory)
  - [Dereferencing](#dereferencing)
- [What Is Borrowing?](#what-is-borrowing)
  - [The Borrowing Analogy](#the-borrowing-analogy)
  - [Borrowing Rules](#borrowing-rules)
- [References as Function Parameters](#references-as-function-parameters)
  - [Before Borrowing (Painful)](#before-borrowing-painful)
  - [After Borrowing (Clean)](#after-borrowing-clean)
  - [Multiple Borrows](#multiple-borrows)
- [References Are Immutable by Default](#references-are-immutable-by-default)
- [The Borrow Checker](#the-borrow-checker)
  - [What the Borrow Checker Enforces](#what-the-borrow-checker-enforces)
  - [Fighting the Borrow Checker](#fighting-the-borrow-checker)
- [References to References](#references-to-references)
- [Dangling References — Prevention](#dangling-references--prevention)
- [References and Ownership — The Full Picture](#references-and-ownership--the-full-picture)
- [Common Patterns](#common-patterns)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Problem That Borrowing Solves

Without borrowing, you have to move values into functions and move them back:

```rust
// ❌ Without borrowing — tedious!
fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)  // Have to give ownership back
}

fn main() {
    let s = String::from("hello");
    let (s, len) = calculate_length(s);  // Move in, get back
    println!("'{}' has length {}", s, len);
}
```

This is awful. Every function that reads a value needs to return it. Imagine doing this with 5 parameters!

```rust
// ✅ With borrowing — clean!
fn calculate_length(s: &String) -> usize {
    s.len()  // Just read it and return the length
}

fn main() {
    let s = String::from("hello");
    let len = calculate_length(&s);  // Lend temporarily
    println!("'{}' has length {}", s, len);  // s is still ours
}
```

---

## What Is a Reference?

A **reference** is a pointer that **borrows** a value without taking ownership. It's created with `&`:

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;  // r is a reference to s
    
    println!("s = {}", s);   // ✅ s is still the owner
    println!("r = {}", r);   // ✅ r can read the data through the reference
}
```

### Creating References

```rust
let x: i32 = 42;
let r: &i32 = &x;    // r is a reference to x

let s: String = String::from("hello");
let r: &String = &s;  // r is a reference to s

let v: Vec<i32> = vec![1, 2, 3];
let r: &Vec<i32> = &v;  // r is a reference to v
```

The `&` operator creates a reference. The type of a reference to `T` is `&T`.

### References in Memory

A reference is stored on the stack as a **pointer** — just the memory address of the original value:

```rust
let s = String::from("hello");
let r = &s;
```

```
Stack                              Heap
┌──────────────────────┐
│ r: &String           │
│   (pointer) ────────────────┐
├──────────────────────┤      │
│ s: String            │      │
│   ptr ───────────────│──────│────→ "hello"
│   len: 5             │ ←────┘
│   cap: 5             │
└──────────────────────┘

r points to s (on the stack).
s points to "hello" (on the heap).
r does NOT point directly to the heap data.
```

A reference is typically **8 bytes** on a 64-bit system (just an address).

### Dereferencing

To get the value a reference points to, use `*` (the dereference operator):

```rust
fn main() {
    let x: i32 = 42;
    let r: &i32 = &x;
    
    println!("r points to: {}", *r);  // Dereference to get 42
    println!("Also works: {}", r);     // println! auto-dereferences
    
    // Explicit comparison
    assert_eq!(*r, 42);
    assert_eq!(*r, x);
}
```

For most uses, Rust automatically dereferences for you (a feature called **deref coercion**). You don't need to write `*` very often.

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;
    
    // These are equivalent — Rust auto-dereferences:
    println!("{}", r.len());     // Auto-deref: calls (*r).len()
    println!("{}", (*r).len());  // Explicit deref (rarely needed)
}
```

---

## What Is Borrowing?

**Borrowing** is the act of creating a reference. When a function takes a reference parameter, we say it **borrows** the value.

### The Borrowing Analogy

```
Ownership is like owning a book:
  - You can read it anytime
  - You can write in it
  - When you're done, you can throw it away (drop it)

Borrowing is like lending your book to a friend:
  - They can read it         (&T — shared/immutable reference)
  - They can't tear out pages (can't modify through &T)
  - They must give it back (reference goes out of scope)
  - You still own it          (original owner retains ownership)
```

### Borrowing Rules

```
╔══════════════════════════════════════════════════════════════╗
║  At any given time, you can have EITHER:                     ║
║                                                              ║
║    • ANY number of immutable references (&T)                ║
║      OR                                                      ║
║    • EXACTLY one mutable reference (&mut T)                 ║
║                                                              ║
║  But NOT both at the same time.                             ║
║                                                              ║
║  References must always be valid (no dangling).             ║
╚══════════════════════════════════════════════════════════════╝
```

In this tutorial, we focus on **immutable references** (`&T`). Mutable references (`&mut T`) are covered in the next tutorial.

---

## References as Function Parameters

### Before Borrowing (Painful)

```rust
fn print_greeting(name: String) {
    println!("Hello, {}!", name);
}

fn print_farewell(name: String) {
    println!("Goodbye, {}!", name);
}

fn main() {
    let name = String::from("Alice");
    print_greeting(name);       // name is moved here
    // print_farewell(name);    // ❌ Can't use name — it was moved!
}
```

### After Borrowing (Clean)

```rust
fn print_greeting(name: &String) {    // Takes a reference
    println!("Hello, {}!", name);
}

fn print_farewell(name: &String) {    // Takes a reference
    println!("Goodbye, {}!", name);
}

fn main() {
    let name = String::from("Alice");
    print_greeting(&name);     // Lend — name is still ours
    print_farewell(&name);     // Lend again — no problem!
    println!("Name: {}", name);  // ✅ Still valid!
}
```

```
Key syntax:
  Function signature: fn foo(s: &String)   ← Takes a reference
  Function call:      foo(&my_string)      ← Creates a reference
```

### Multiple Borrows

You can have **multiple immutable references** at the same time:

```rust
fn main() {
    let s = String::from("hello");
    
    let r1 = &s;   // ✅ First borrow
    let r2 = &s;   // ✅ Second borrow
    let r3 = &s;   // ✅ Third borrow
    
    println!("{} {} {}", r1, r2, r3);  // ✅ All valid simultaneously
    
    // This is safe because NONE of them can modify s
    // Multiple readers, no writers → no data race possible
}
```

This is the "many readers" part of the rule. Since no one can modify the data, any number of references can read safely.

---

## References Are Immutable by Default

An immutable reference (`&T`) gives **read-only access**. You cannot modify the data through it:

```rust
fn add_exclamation(s: &String) {
    // s.push_str("!");  // ❌ ERROR: cannot borrow `*s` as mutable
    println!("{}", s);   // ✅ Can read, no problem
}

fn main() {
    let s = String::from("hello");
    add_exclamation(&s);
    println!("{}", s);  // Still "hello" — not modified
}
```

The compiler error:

```
error[E0596]: cannot borrow `*s` as mutable, as it is behind a `&` reference
 --> src/main.rs:2:5
  |
1 | fn add_exclamation(s: &String) {
  |                       ------- help: consider changing this to be mutable: `&mut String`
2 |     s.push_str("!");
  |     ^ `s` is a `&` reference, so the data it refers to cannot be borrowed as mutable
```

The compiler even tells you how to fix it: use `&mut String`. We'll cover that in the next tutorial.

---

## The Borrow Checker

The **borrow checker** is a part of the Rust compiler that enforces the borrowing rules at compile time. It ensures:

1. **References are always valid** — no dangling pointers
2. **Shared access is read-only** — immutable references can't modify data
3. **Exclusive access for mutation** — only one mutable reference at a time

### What the Borrow Checker Enforces

```rust
// RULE: Can't move the owner while references exist
fn main() {
    let s = String::from("hello");
    let r = &s;       // Borrow s
    
    // let s2 = s;    // ❌ ERROR: can't move s while r borrows it
    
    println!("{}", r);  // r is used here — s must stay alive
}
```

Why? If `s` were moved (and its data freed), `r` would be a **dangling pointer** — pointing to freed memory. The borrow checker prevents this.

```rust
// RULE: Can't modify while immutable references exist
fn main() {
    let mut s = String::from("hello");
    let r = &s;        // Immutable borrow
    
    // s.push_str("!");  // ❌ Can't modify — r might see the change!
    
    println!("{}", r);  // r is used here
    
    // After r is no longer used, we CAN modify s again
    s.push_str("!");    // ✅ r is done — no more borrows
    println!("{}", s);  // "hello!"
}
```

### Non-Lexical Lifetimes (NLL)

Modern Rust is smart about when borrows end. A borrow ends at the **last point the reference is used**, not necessarily at the end of the scope:

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r = &s;
    println!("{}", r);  // ← Last use of r
    // r's borrow effectively ends here
    
    s.push_str(" world");  // ✅ No active borrows — modification OK
    println!("{}", s);
}
```

Before NLL (Rust <1.31), this wouldn't have compiled because `r` existed until the end of the scope. With NLL, the compiler tracks where references are actually used.

### Fighting the Borrow Checker

When the borrow checker rejects your code, it's usually telling you about a real potential bug. Common resolutions:

1. **Restructure code** — move the borrow to a smaller scope
2. **Clone** — if borrowing is too complex, clone the data
3. **Use different data structures** — sometimes you need `Rc`, `RefCell`, etc.

```rust
// ❌ Borrow checker error
fn main() {
    let mut data = vec![1, 2, 3];
    let first = &data[0];        // Immutable borrow of data
    data.push(4);                // ❌ Mutable borrow while first exists
    println!("{}", first);
}

// ✅ Fix: finish using the reference before modifying
fn main() {
    let mut data = vec![1, 2, 3];
    let first = data[0];         // Copy the value (i32 is Copy)
    data.push(4);                // ✅ No borrow active
    println!("{}", first);       // Prints 1
}
```

---

## References to References

You can have references to references (multiple levels of indirection):

```rust
fn main() {
    let x: i32 = 42;
    let r1: &i32 = &x;            // Reference to x
    let r2: &&i32 = &r1;          // Reference to a reference
    let r3: &&&i32 = &r2;         // Three levels deep
    
    // Rust auto-dereferences through all levels:
    println!("{}", r3);     // Prints 42
    println!("{}", **r2);   // Explicit: dereference twice
    println!("{}", ***r3);  // Explicit: dereference three times
}
```

In practice, you rarely go more than one level deep. Rust's auto-deref makes it seamless.

---

## Dangling References — Prevention

A **dangling reference** is a reference that points to memory that has been freed. This is a common source of bugs in C/C++. Rust prevents it completely:

```rust
// ❌ This won't compile — Rust catches the dangling reference
// fn dangle() -> &String {
//     let s = String::from("hello");
//     &s  // Return a reference to s
// }  // s is dropped here — the reference would point to freed memory!

// error[E0106]: missing lifetime specifier
// This function's return type contains a borrowed value,
// but there is no value for it to be borrowed from.
```

The fix: return the owned value instead:

```rust
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // Move ownership to the caller — no dangling reference
}

fn main() {
    let s = no_dangle();
    println!("{}", s);  // ✅ 
}
```

### Why Dangling References Are Impossible

```
Dangling reference scenario (prevented by Rust):

1. Function creates a String         s → "hello" (heap)
2. Function returns &s               → &s points to s's data
3. Function returns → s is dropped   → heap freed!
4. Caller uses &s                    → DANGLING! Points to freed memory

Rust's solution: The compiler REFUSES to compile step 2.
It sees that s will be dropped and the reference would dangle.
```

---

## References and Ownership — The Full Picture

Let's put it all together:

```rust
fn main() {
    let s = String::from("hello");  // s owns the String
    
    // Any number of immutable references are OK:
    let r1 = &s;
    let r2 = &s;
    println!("{} {} {}", s, r1, r2);  // Owner and borrows coexist
    
    // The owner can't be moved while borrows are active:
    // let s2 = s;  // ❌ r1 and r2 borrow s
    
    // After all references are done, we can reclaim full control:
    let r1 = &s;
    println!("{}", r1);               // Last use of r1
    
    println!("{}", s);                // ✅ No borrows active
}
```

### Ownership vs Borrowing — Decision Guide

```
Do I need to:

Read the data?
  → Use &T (immutable reference)

Modify the data?
  → Use &mut T (mutable reference — next tutorial)

Take the data permanently?
  → Take ownership (T)

Store the data somewhere that outlives the current function?
  → Take ownership (T) or clone
```

---

## Common Patterns

### Pattern 1: Read-only Function Parameter

The most common use of references — a function that reads but doesn't modify:

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }
    &s[..]
}

fn main() {
    let s = String::from("hello world");
    let word = first_word(&s);
    println!("First word: {}", word);  // "hello"
}
```

### Pattern 2: Multiple Read-Only References

```rust
fn longest(a: &str, b: &str) -> &str {
    // (This actually needs lifetime annotations — simplified here)
    if a.len() >= b.len() { a } else { b }
}
```

### Pattern 3: Borrowing in Loops

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Charlie"];
    
    for name in &names {  // Borrow each element
        println!("Hello, {}!", name);
    }
    
    // names is still valid after the loop
    println!("Total: {}", names.len());
}
```

### Pattern 4: Method Calls on References

```rust
fn main() {
    let s = String::from("Hello, World!");
    let r = &s;
    
    // Method calls work through references (auto-deref):
    println!("Length: {}", r.len());           // 13
    println!("Uppercase: {}", r.to_uppercase()); // "HELLO, WORLD!"
    println!("Contains: {}", r.contains("World")); // true
}
```

### Pattern 5: Prefer &str Over &String

```rust
// ❌ Less flexible — only accepts &String
fn greet(name: &String) {
    println!("Hello, {}!", name);
}

// ✅ More flexible — accepts &String AND &str
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let owned = String::from("Alice");
    let literal = "Bob";
    
    greet(&owned);    // ✅ &String coerces to &str
    greet(literal);   // ✅ &str works directly
}
```

We'll explore this `String` vs `&str` difference in detail in tutorial 8.

---

## Common Mistakes

### Mistake 1: Returning a Reference to a Local Variable

```rust
// ❌ Dangling reference — won't compile
// fn create() -> &String {
//     let s = String::from("hello");
//     &s
// }

// ✅ Return the owned value instead
fn create() -> String {
    String::from("hello")
}
```

### Mistake 2: Moving While Borrowed

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;
    
    // let s2 = s;    // ❌ Can't move s while r borrows it
    
    println!("{}", r);  // Still using the borrow
    
    // After r's last use, we CAN move s:
    let s2 = s;         // ✅ r is no longer used
    println!("{}", s2);
}
```

### Mistake 3: Modifying Through an Immutable Reference

```rust
fn try_modify(s: &String) {
    // s.push_str("!");  // ❌ Cannot modify through &String
}

// You need &mut String to modify (next tutorial)
```

### Mistake 4: Forgetting the & in the Function Call

```rust
fn print_len(s: &String) {
    println!("Length: {}", s.len());
}

fn main() {
    let s = String::from("hello");
    
    // print_len(s);     // ❌ Type mismatch: expected &String, got String
    print_len(&s);       // ✅ Create a reference with &
}
```

### Mistake 5: Expecting References to Extend Lifetime

```rust
fn main() {
    let r;
    {
        let s = String::from("hello");
        r = &s;
    }  // s is dropped here
    
    // println!("{}", r);  // ❌ r is a dangling reference!
}
```

A reference cannot keep a value alive. The value's lifetime is determined by its owner's scope.

---

## Exercises

### Exercise 1: Convert to Borrowing

Rewrite this code to use borrowing so you don't need to return the String back:

```rust
fn count_vowels(s: String) -> (String, usize) {
    let count = s.chars()
        .filter(|c| "aeiouAEIOU".contains(*c))
        .count();
    (s, count)
}

fn main() {
    let text = String::from("Hello World");
    let (text, vowels) = count_vowels(text);
    println!("'{}' has {} vowels", text, vowels);
}
```

<details>
<summary>Answer</summary>

```rust
fn count_vowels(s: &String) -> usize {
    s.chars()
        .filter(|c| "aeiouAEIOU".contains(*c))
        .count()
}

fn main() {
    let text = String::from("Hello World");
    let vowels = count_vowels(&text);
    println!("'{}' has {} vowels", text, vowels);
}
```

Even better, take `&str`:
```rust
fn count_vowels(s: &str) -> usize { ... }
```

</details>

### Exercise 2: Multiple References

Will this compile? Why or why not?

```rust
fn main() {
    let s = String::from("data");
    let a = &s;
    let b = &s;
    let c = &s;
    println!("{} {} {} {}", s, a, b, c);
}
```

### Exercise 3: Find the Bug

What's wrong with this code?

```rust
fn main() {
    let r;
    {
        let s = String::from("hello");
        r = &s;
    }
    println!("{}", r);
}
```

### Exercise 4: Borrow in a Function

Write a function `is_long` that takes a `&String` and returns `true` if the string length is greater than 10. The original string should remain usable after the function call.

### Exercise 5: Multiple Borrows in Practice

Write a function `compare_lengths` that takes two `&String` parameters and prints which one is longer (or if they're equal). Call it multiple times with the same strings to demonstrate that borrowing allows reuse.

---

## Summary

### Reference Syntax

| Syntax | Meaning |
|--------|---------|
| `&x` | Create an immutable reference to `x` |
| `&T` | Type of an immutable reference to `T` |
| `*r` | Dereference a reference (get the value) |
| `&String` | Immutable reference to a String |
| `&str` | Immutable reference to a string slice |
| `&[T]` | Immutable reference to a slice |

### Borrowing Rules (for Immutable References)

| Rule | Description |
|------|-------------|
| **Multiple OK** | You can have any number of `&T` at the same time |
| **Read-only** | You cannot modify data through `&T` |
| **Valid only** | References must always point to valid data |
| **No move during borrow** | Owner can't be moved while references exist |
| **NLL** | Borrows end at last use, not end of scope |

### When to Use References

| Situation | Use |
|-----------|-----|
| Function needs to read data | `&T` |
| Function needs to modify data | `&mut T` (next tutorial) |
| Function needs to own the data | `T` (take ownership) |
| Read-only iteration | `for item in &collection` |

### Key Takeaways

1. **A reference (`&T`) borrows a value without taking ownership**
2. **References are just pointers** — 8 bytes on the stack
3. **Immutable references allow reading but not modifying**
4. **Multiple immutable references are allowed simultaneously**
5. **You can't move the owner while borrows exist**
6. **The borrow checker prevents dangling references at compile time**
7. **Prefer `&str` over `&String`** for function parameters
8. **NLL makes the borrow checker smarter** — borrows end at last use

---

## What's Next?

We've been reading data through references. But what about modifying it? Time to learn about `&mut T` — mutable references.

**Next Tutorial:** [Mutable References →](./06-mutable-references.md)

---

<p align="center">
  <i>Tutorial 5 of 8 — Stage 3: Ownership</i>
</p>

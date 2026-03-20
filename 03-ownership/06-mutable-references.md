# Mutable References 🔓

> **Mutable references give you the power to modify borrowed data — but with a strict rule: only ONE mutable reference at a time. This prevents data races at compile time.**

---

## Table of Contents

- [The Problem: Modifying Borrowed Data](#the-problem-modifying-borrowed-data)
- [Creating Mutable References](#creating-mutable-references)
  - [The Syntax](#the-syntax)
  - [Requirements for &mut](#requirements-for-mut)
- [The One Mutable Reference Rule](#the-one-mutable-reference-rule)
  - [Why Only One?](#why-only-one)
  - [Data Races Explained](#data-races-explained)
- [The Either/Or Rule](#the-eitheror-rule)
  - [Many Readers OR One Writer](#many-readers-or-one-writer)
  - [Why Not Both?](#why-not-both)
  - [The Iterator Invalidation Problem](#the-iterator-invalidation-problem)
- [Mutable References in Functions](#mutable-references-in-functions)
- [Reborrowing](#reborrowing)
- [Non-Lexical Lifetimes with &mut](#non-lexical-lifetimes-with-mut)
- [Mutable References and Structs](#mutable-references-and-structs)
- [Common Patterns](#common-patterns)
- [The Borrow Checker in Action](#the-borrow-checker-in-action)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Problem: Modifying Borrowed Data

Immutable references (`&T`) let you read data. But sometimes a function needs to **modify** data without taking ownership:

```rust
// ❌ Immutable reference can't modify
fn add_exclamation(s: &String) {
    // s.push_str("!");  // ERROR: cannot borrow as mutable
}

// ❌ Taking ownership is wasteful — you'd have to return it
fn add_exclamation(s: String) -> String {
    let mut s = s;
    s.push_str("!");
    s  // Have to give it back
}

// ✅ Mutable reference — modify without taking ownership
fn add_exclamation(s: &mut String) {
    s.push_str("!");
}
```

---

## Creating Mutable References

### The Syntax

```rust
fn main() {
    let mut s = String::from("hello");  // Variable must be mut
    
    let r = &mut s;  // Create a mutable reference
    
    r.push_str(" world");  // Modify through the reference
    
    println!("{}", r);  // "hello world"
}
```

Three things must align:

```
1. The variable must be declared `mut`:     let mut s = ...
2. The reference must be `&mut`:            let r = &mut s;
3. The parameter type must be `&mut T`:     fn foo(s: &mut String)
```

### Requirements for `&mut`

```rust
// ❌ Can't create &mut of a non-mut variable
let s = String::from("hello");   // Not mut!
// let r = &mut s;               // ERROR: cannot borrow as mutable

// ✅ Variable must be mut
let mut s = String::from("hello");
let r = &mut s;
r.push_str("!");
```

The compiler error:

```
error[E0596]: cannot borrow `s` as mutable, as it is not declared as mutable
 --> src/main.rs:3:13
  |
2 |     let s = String::from("hello");
  |         - help: consider changing this to be mutable: `mut s`
3 |     let r = &mut s;
  |             ^^^^^^ cannot borrow as mutable
```

---

## The One Mutable Reference Rule

**At any given time, you can have exactly ONE mutable reference to a particular piece of data.**

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &mut s;
    // let r2 = &mut s;  // ❌ ERROR: cannot borrow s as mutable more than once
    
    r1.push_str("!");
    println!("{}", r1);
}
```

The compiler error:

```
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:4:14
  |
3 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
4 |     let r2 = &mut s;
  |              ^^^^^^ second mutable borrow occurs here
5 |     r1.push_str("!");
  |     -- first borrow later used here
```

### Why Only One?

This prevents **data races** — one of the most dangerous bugs in concurrent and even single-threaded programming.

### Data Races Explained

A **data race** occurs when:
1. Two or more pointers access the same data at the same time
2. At least one of them is writing
3. There's no synchronization

```
Scenario: Two mutable references to the same Vec

Thread/Code A:          Thread/Code B:
  vec.push(4)             vec.remove(0)
       ↓                       ↓
  Allocates new           Shifts elements
  memory, copies          left, changes
  elements...             length...
       ↓                       ↓
  CRASH! 💥 — B changed the Vec while A was using it

Even in single-threaded code, this is dangerous:
  for item in &vec {    // Reading...
      vec.push(0);      // ...while modifying? → undefined behavior in C++!
  }
```

Rust's rule makes this impossible:
- If someone is writing (`&mut T`), nobody else can read or write
- If people are reading (`&T`), nobody can write

### Visualizing the Rule

```
✅ Multiple readers (no writers):
    [ Reader 1 ] ──→ [  Data  ] ←── [ Reader 2 ]
                          ↑
                     [ Reader 3 ]

✅ Exactly one writer (no readers):
    [ Writer ] ──→ [  Data  ]

❌ NEVER: readers AND writers simultaneously:
    [ Reader ] ──→ [  Data  ] ←── [ Writer ]  ← FORBIDDEN!

❌ NEVER: multiple writers:
    [ Writer 1 ] ──→ [  Data  ] ←── [ Writer 2 ]  ← FORBIDDEN!
```

---

## The Either/Or Rule

### Many Readers OR One Writer

This is the **fundamental rule** of Rust's borrowing system:

```rust
fn main() {
    let mut s = String::from("hello");
    
    // Option A: Many immutable references ✅
    {
        let r1 = &s;
        let r2 = &s;
        let r3 = &s;
        println!("{} {} {}", r1, r2, r3);
    }
    
    // Option B: One mutable reference ✅
    {
        let r = &mut s;
        r.push_str(" world");
        println!("{}", r);
    }
    
    // Option C: Mix of both ❌
    // {
    //     let r1 = &s;        // Immutable borrow
    //     let r2 = &mut s;    // ❌ Can't have &mut while & exists
    //     println!("{}", r1);
    // }
}
```

### Why Not Both?

Consider what would happen if you could have `&T` and `&mut T` at the same time:

```rust
// HYPOTHETICAL: if Rust allowed this (it doesn't!)
let mut data = vec![1, 2, 3];
let reference = &data[0];  // reference points to the first element
data.push(4);              // Vec might reallocate to grow!
println!("{}", reference); // ← reference might point to freed memory!
```

```
Step 1: data = [1, 2, 3] at address 0x1000
        reference = &data[0] → points to 0x1000

Step 2: data.push(4) → needs more space!
        Allocates new memory at 0x2000
        Copies [1, 2, 3, 4] to 0x2000
        Frees old memory at 0x1000!

Step 3: reference still points to 0x1000 → DANGLING POINTER! 💥
```

Rust prevents this at compile time:

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let reference = &data[0];  // Immutable borrow
    // data.push(4);           // ❌ Can't mutate while immutably borrowed
    println!("{}", reference);
    
    data.push(4);              // ✅ After reference is done
    println!("{:?}", data);    // [1, 2, 3, 4]
}
```

### The Iterator Invalidation Problem

This is a classic bug in C++ and Java. Rust prevents it entirely:

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];
    
    // Iterating (immutable borrow of numbers)
    // while also trying to modify (mutable borrow)
    // for num in &numbers {
    //     if *num % 2 == 0 {
    //         numbers.push(*num * 10);  // ❌ Can't modify while iterating!
    //     }
    // }
    
    // ✅ Collect modifications, then apply
    let to_add: Vec<i32> = numbers.iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * 10)
        .collect();
    
    numbers.extend(to_add);
    println!("{:?}", numbers);  // [1, 2, 3, 4, 5, 20, 40]
}
```

---

## Mutable References in Functions

```rust
fn append_greeting(name: &mut String) {
    name.push_str(", welcome!");
}

fn double_all(numbers: &mut Vec<i32>) {
    for num in numbers.iter_mut() {
        *num *= 2;
    }
}

fn main() {
    let mut name = String::from("Alice");
    append_greeting(&mut name);
    println!("{}", name);  // "Alice, welcome!"
    
    let mut nums = vec![1, 2, 3, 4, 5];
    double_all(&mut nums);
    println!("{:?}", nums);  // [2, 4, 6, 8, 10]
}
```

Function signature guide:

```
fn read_only(s: &String)      → Borrows, read-only
fn modify(s: &mut String)     → Borrows, can modify
fn consume(s: String)          → Takes ownership
```

---

## Reborrowing

When you have a `&mut T`, you can temporarily create a new reference from it. This is called **reborrowing**:

```rust
fn print_and_modify(s: &mut String) {
    // Reborrowing: creating &String from &mut String
    let read_ref: &String = &*s;   // Can also just use s in a read context
    println!("Current: {}", read_ref);
    
    // Or just use the &mut directly—Rust auto-reborrows:
    println!("Length: {}", s.len());  // Implicit reborrow as &String
    
    s.push_str("!");  // Use the &mut again
}
```

Reborrowing to pass to another function:

```rust
fn log_value(s: &String) {
    println!("[LOG] {}", s);
}

fn process(s: &mut String) {
    log_value(s);     // Implicit reborrow: &mut String → &String
    s.push_str("!");
    log_value(s);     // Reborrow again
}

fn main() {
    let mut s = String::from("hello");
    process(&mut s);
    println!("{}", s);  // "hello!"
}
```

---

## Non-Lexical Lifetimes with `&mut`

NLL lets you use mutable and immutable references as long as they don't overlap:

```rust
fn main() {
    let mut s = String::from("hello");
    
    // Phase 1: Multiple readers
    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // r1 and r2 are no longer used — borrows end
    
    // Phase 2: One writer
    let r3 = &mut s;
    r3.push_str(" world");
    println!("{}", r3);
    // r3 is no longer used — borrow ends
    
    // Phase 3: Back to reading
    println!("{}", s);
}
```

```
Timeline:
  r1 ───────── used ──── done
  r2 ───────── used ──── done
                              r3 ──── used ──── done
  |─── immutable borrows ───|──── mutable ────|── owner ──|
  
  No overlap between immutable and mutable borrows ✅
```

But this fails:

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;           // Immutable borrow starts
    let r2 = &mut s;       // ❌ Can't start mutable borrow while r1 exists
    println!("{}", r1);    // r1 is still being used!
}
```

---

## Mutable References and Structs

```rust
struct Player {
    name: String,
    health: i32,
    score: u32,
}

fn take_damage(player: &mut Player, damage: i32) {
    player.health -= damage;
    if player.health < 0 {
        player.health = 0;
    }
    println!("{} took {} damage, health: {}", player.name, damage, player.health);
}

fn add_score(player: &mut Player, points: u32) {
    player.score += points;
    println!("{} earned {} points, total: {}", player.name, points, player.score);
}

fn display(player: &Player) {  // Read-only borrow
    println!("Player: {} | HP: {} | Score: {}", player.name, player.health, player.score);
}

fn main() {
    let mut hero = Player {
        name: String::from("Hero"),
        health: 100,
        score: 0,
    };
    
    display(&hero);
    take_damage(&mut hero, 30);
    add_score(&mut hero, 100);
    take_damage(&mut hero, 25);
    add_score(&mut hero, 50);
    display(&hero);
}
```

---

## Common Patterns

### Pattern 1: Modify and Return Information

```rust
fn pop_and_report(items: &mut Vec<String>) -> Option<String> {
    let popped = items.pop();
    if let Some(ref item) = popped {
        println!("Removed: {} ({} remaining)", item, items.len());
    }
    popped
}
```

### Pattern 2: Swap Values

```rust
fn main() {
    let mut a = String::from("hello");
    let mut b = String::from("world");
    
    std::mem::swap(&mut a, &mut b);
    
    println!("a = {}", a);  // "world"
    println!("b = {}", b);  // "hello"
}
```

### Pattern 3: Scoped Mutable Borrow

Use a block to limit the scope of a mutable borrow:

```rust
fn main() {
    let mut data = vec![3, 1, 4, 1, 5, 9];
    
    // Mutable borrow in a limited scope
    {
        let slice = &mut data[..];
        slice.sort();
    }  // Mutable borrow ends here
    
    // Can now read
    println!("{:?}", data);  // [1, 1, 3, 4, 5, 9]
}
```

### Pattern 4: In-Place Modification

```rust
fn capitalize_first(s: &mut String) {
    if let Some(first) = s.get(0..1) {
        let upper = first.to_uppercase();
        s.replace_range(0..1, &upper);
    }
}

fn main() {
    let mut greeting = String::from("hello world");
    capitalize_first(&mut greeting);
    println!("{}", greeting);  // "Hello world"
}
```

### Pattern 5: Building Results In-Place

```rust
fn collect_evens(source: &[i32], dest: &mut Vec<i32>) {
    for &num in source {
        if num % 2 == 0 {
            dest.push(num);
        }
    }
}

fn main() {
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let mut evens = Vec::new();
    
    collect_evens(&numbers, &mut evens);
    println!("{:?}", evens);  // [2, 4, 6, 8, 10]
}
```

---

## The Borrow Checker in Action

Let's walk through several scenarios:

### Scenario 1: Sequential Borrows ✅

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;
    println!("{}", r1);  // r1's last use
    
    let r2 = &mut s;     // ✅ r1 is done
    r2.push_str("!");
    println!("{}", r2);  // r2's last use
    
    println!("{}", s);   // ✅ No borrows active
}
```

### Scenario 2: Overlapping Borrows ❌

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;         // Immutable borrow
    let r2 = &mut s;     // ❌ Mutable borrow while r1 exists
    
    println!("{}", r1);  // r1 is still alive!
}
```

### Scenario 3: Borrow of Struct Fields ✅

Rust can track borrows of individual struct fields separately:

```rust
struct Pair {
    first: String,
    second: String,
}

fn main() {
    let mut pair = Pair {
        first: String::from("hello"),
        second: String::from("world"),
    };
    
    let r1 = &pair.first;       // Borrow first field
    let r2 = &mut pair.second;  // ✅ Borrow different field — OK!
    
    r2.push_str("!");
    println!("{} {}", r1, r2);
}
```

But you can't borrow the whole struct AND a field:

```rust
// let whole = &mut pair;       // ❌ Can't borrow whole struct
// let field = &pair.first;     //    while also borrowing a field
```

---

## Common Mistakes

### Mistake 1: Forgetting `mut` on the Variable

```rust
let s = String::from("hello");
// let r = &mut s;  // ❌ s is not declared as mut

let mut s = String::from("hello");
let r = &mut s;  // ✅
```

### Mistake 2: Multiple Mutable References

```rust
let mut s = String::from("hello");
let r1 = &mut s;
// let r2 = &mut s;  // ❌ Only one &mut at a time
r1.push_str("!");
```

### Mistake 3: Mixing & and &mut

```rust
let mut s = String::from("hello");
let r1 = &s;       // Immutable borrow
// let r2 = &mut s;  // ❌ Can't have &mut while & exists
println!("{}", r1); // r1 is still in use
```

Fix: finish using `r1` first:

```rust
let mut s = String::from("hello");
let r1 = &s;
println!("{}", r1);  // Last use of r1

let r2 = &mut s;     // ✅ r1 is done
r2.push_str("!");
```

### Mistake 4: Calling &mut Methods While Borrowing

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];     // Immutable borrow
// v.push(4);           // ❌ push takes &mut self
println!("{}", first);

v.push(4);              // ✅ After first is done
```

### Mistake 5: Returning &mut to Local Data

```rust
// ❌ Can't return &mut to local data
// fn create() -> &mut String {
//     let mut s = String::from("hello");
//     &mut s  // s is about to be dropped!
// }

// ✅ Return the owned value
fn create() -> String {
    let mut s = String::from("hello");
    s.push_str("!");
    s
}
```

---

## Exercises

### Exercise 1: Write a Mutating Function

Write a function `make_uppercase(s: &mut String)` that converts the string to uppercase in-place. Test it:

```rust
fn main() {
    let mut text = String::from("hello world");
    make_uppercase(&mut text);
    assert_eq!(text, "HELLO WORLD");
    println!("{}", text);
}
```

### Exercise 2: Will It Compile?

For each program, predict if it compiles:

**A:**
```rust
fn main() {
    let mut x = 5;
    let r = &mut x;
    *r += 1;
    println!("{}", x);
}
```

**B:**
```rust
fn main() {
    let mut s = String::from("hello");
    let r1 = &s;
    let r2 = &mut s;
    println!("{} {}", r1, r2);
}
```

**C:**
```rust
fn main() {
    let mut s = String::from("hello");
    let r1 = &s;
    println!("{}", r1);
    let r2 = &mut s;
    r2.push_str("!");
    println!("{}", r2);
}
```

<details>
<summary>Answers</summary>

- **A:** ✅ Compiles. `r` is the only borrow. After `r`'s last use, `x` can be printed.
- **B:** ❌ Error. Can't have `&s` and `&mut s` at the same time (both used in `println!`).
- **C:** ✅ Compiles with NLL. `r1` is last used before `r2` is created — no overlap.

</details>

### Exercise 3: Fix the Borrow Error

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];
    let first = &numbers[0];
    numbers.push(6);
    println!("First: {}", first);
}
```

### Exercise 4: Implement `remove_negatives`

Write a function that takes `&mut Vec<i32>` and removes all negative numbers:

```rust
fn remove_negatives(nums: &mut Vec<i32>) {
    // Your code here
}

fn main() {
    let mut data = vec![3, -1, 4, -1, 5, -9, 2, 6];
    remove_negatives(&mut data);
    println!("{:?}", data);  // [3, 4, 5, 2, 6]
}
```

---

## Summary

### Mutable Reference Syntax

| Syntax | Meaning |
|--------|---------|
| `&mut x` | Create a mutable reference to `x` |
| `&mut T` | Type of a mutable reference to `T` |
| `*r = value` | Write through a mutable reference |
| `fn f(x: &mut T)` | Function takes a mutable borrow |
| `f(&mut x)` | Pass a mutable reference to a function |

### The Complete Borrowing Rules

```
At any given time, you can have EITHER:
  ✅ Any number of &T (immutable references)
  OR
  ✅ Exactly one &mut T (mutable reference)
  
  NEVER both at the same time.
  NEVER multiple &mut T at the same time.
  References must always be valid.
```

### Reference Comparison

| Feature | `&T` (immutable) | `&mut T` (mutable) |
|---------|-------------------|---------------------|
| Read data | ✅ Yes | ✅ Yes |
| Modify data | ❌ No | ✅ Yes |
| Multiple at same time | ✅ Yes (unlimited) | ❌ No (exactly one) |
| Coexist with `&T`? | ✅ Yes (with other `&T`) | ❌ No |
| Variable must be `mut`? | No | Yes |

### Key Takeaways

1. **`&mut T` gives read AND write access** to borrowed data
2. **Only ONE `&mut T` at a time** — prevents data races
3. **Can't mix `&T` and `&mut T`** — readers and writers conflict
4. **The variable must be `mut`** to create a `&mut` reference
5. **NLL means borrows end at last use** — not end of scope
6. **Struct fields can be borrowed independently**
7. **The borrow checker catches bugs at compile time** that other languages miss at runtime

---

## What's Next?

We've covered ownership, moves, cloning, immutable references, and mutable references. Now let's learn about **slices** — a way to reference a contiguous portion of a collection.

**Next Tutorial:** [Slices →](./07-slices.md)

---

<p align="center">
  <i>Tutorial 6 of 8 — Stage 3: Ownership</i>
</p>

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

### The Data Race Problem — Why One Mutable Reference Matters

To truly appreciate Rust's rule, you need to understand what data races are **at the hardware level** — and why every other language struggles with them.

#### What Happens at the Hardware Level

Modern CPUs have multiple cores, each with their own cache hierarchy (L1, L2, L3). When two cores write to the same **cache line** (typically 64 bytes) simultaneously, the hardware must arbitrate:

```
  Core 0                     Core 1
  ┌──────────┐               ┌──────────┐
  │ L1 Cache │               │ L1 Cache │
  │ [x = 42] │               │ [x = 99] │
  └────┬─────┘               └────┬─────┘
       │  WRITE x = 42             │  WRITE x = 99
       └──────────┐   ┌───────────┘
                  ↓   ↓
            ┌──────────────┐
            │  Shared RAM   │
            │  x = ???      │  ← Which write wins? UNDEFINED.
            └──────────────┘
```

The CPU cache coherence protocol (MESI/MOESI) tries to keep caches consistent, but when two writes happen in the **same clock cycle window**, the result is genuinely undefined — not just "one or the other wins," but potentially a **torn read** where half the bytes come from one write and half from another.

#### The TOCTOU Problem (Time of Check to Time of Use)

A particularly insidious race condition pattern:

```
Thread A:                           Thread B:
  1. if balance >= 100 {            1. if balance >= 100 {
     // balance is 150, ok!            // balance is 150, ok!
  2.   balance -= 100;              2.   balance -= 100;
     // balance is now 50               // balance is now -50! 💥
  }
```

Between **checking** a value and **acting** on it, another thread can change it. This is called TOCTOU — and it's responsible for countless real-world bugs.

#### Famous Data Race Disasters

- **Therac-25 (1985-1987):** A radiation therapy machine had a race condition between its UI and beam control. Due to unsynchronized shared state, patients received radiation doses **100x the intended amount**. At least 3 people died and 3 more were seriously injured.
- **Northeast Blackout of 2003:** A race condition in the alarm system software at FirstEnergy caused alarms to silently fail. Operators had no warning as the grid cascaded into failure. 55 million people lost power.

These aren't hypothetical — **data races kill people and shut down infrastructure.**

#### How Other Languages "Solve" Data Races

| Language | Data Race Prevention | When Checked | Runtime Cost | Escape Hatches |
|----------|---------------------|-------------|-------------|----------------|
| **Rust** | One-mutable-reference rule | Compile time | Zero | `unsafe` blocks |
| **Java** | `synchronized`, `volatile`, `j.u.c` locks | Runtime | Lock overhead on every access | Everything outside `synchronized` |
| **Go** | Channels, goroutines | Runtime (race detector optional) | Channel overhead | Shared memory still possible |
| **Python** | GIL (Global Interpreter Lock) | Runtime | Prevents true parallelism entirely | C extensions bypass GIL |
| **C/C++** | `mutex`, `atomic` (programmer responsibility) | Never (undefined behavior) | Lock/atomic overhead | Everything — races are UB |

- **Java** requires you to remember to use `synchronized` on every access. Miss one? Silent data corruption.
- **Go** says "share memory by communicating" but the `-race` detector is opt-in and only catches races that actually execute during your test run.
- **Python's GIL** is the bluntest possible hammer — it prevents all true parallelism in CPython, and they're only now (2024-2026) experimenting with removing it.
- **C/C++** data races are literally **undefined behavior** — the compiler can assume they never happen, which means optimizations can make the bug even worse.

Rust's approach is unique: the one-mutable-reference rule makes data races **unrepresentable** in safe code. You can't even write the buggy program — the compiler rejects it. This is enforced entirely at compile time, so there is **zero runtime overhead**. No locks, no channels, no GIL — just static proof that your code is race-free.

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

### Interior Mutability — When the Rules Aren't Enough

Sometimes you have a legitimate need: multiple parts of your code hold references to the same data, **and** one of them needs to mutate it. The "aliasing XOR mutability" rule is too strict for these patterns.

Rust's answer: **interior mutability** — types that allow mutation through an **immutable reference** (`&self`), moving the borrow check from compile time to runtime.

#### The Interior Mutability Types

```
┌──────────────────────────────────────────────────────────────┐
│                  Interior Mutability Toolkit                  │
├──────────────┬──────────────┬────────────────────────────────┤
│   Cell<T>    │  RefCell<T>  │        Mutex<T> / RwLock<T>    │
├──────────────┼──────────────┼────────────────────────────────┤
│ Copy types   │ Any type     │ Any type + Send                │
│ get()/set()  │ borrow()/    │ lock()/read()/write()          │
│              │ borrow_mut() │                                │
│ No refs to   │ Runtime      │ OS-level locking               │
│ interior     │ borrow check │ Thread-safe                    │
│ Zero-cost    │ Small cost   │ Higher cost                    │
│ !Sync        │ !Sync        │ Sync (whole point)             │
└──────────────┴──────────────┴────────────────────────────────┘
```

- **`Cell<T>`**: For `Copy` types only. No references to the interior — you `get()` a copy and `set()` a new value. Essentially free, no runtime borrow tracking.
- **`RefCell<T>`**: For any type. Tracks borrows at runtime via `borrow()` (returns `Ref<T>`) and `borrow_mut()` (returns `RefMut<T>`). **Panics if you violate borrowing rules** — you're trading a compile-time error for a runtime panic.
- **`Mutex<T>`**: Like `RefCell` but thread-safe. Uses OS-level locking so it can be shared across threads (`Sync`). The `lock()` call blocks until the mutex is available.

#### RefCell in Action

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);

    // Multiple code paths share &data (immutable reference)
    // but can still mutate through borrow_mut()
    {
        let mut inner = data.borrow_mut();  // Runtime checked
        inner.push(4);
    }  // RefMut dropped — borrow released

    // Now we can borrow immutably
    println!("{:?}", data.borrow());  // [1, 2, 3, 4]

    // ⚠️ THIS PANICS at runtime:
    // let r1 = data.borrow();
    // let r2 = data.borrow_mut();  // PANIC: already borrowed!
}
```

#### When to Use Interior Mutability

| Pattern | Why You Need It | Which Type |
|---------|----------------|------------|
| Observer/callback lists | Multiple observers hold `&self`, need to register/unregister | `RefCell<Vec<T>>` |
| Caching / memoization | `&self` method wants to cache results | `Cell<Option<T>>` or `RefCell<HashMap>` |
| Graph data structures | Nodes reference each other, need mutation | `RefCell<T>` with `Rc<T>` |
| Shared config across threads | Multiple threads read, occasional write | `Mutex<T>` or `RwLock<T>` |
| Mock objects in tests | Test injects `&dyn Trait`, mock needs to record calls | `RefCell<Vec<Call>>` |

> **Important:** Interior mutability is **NOT** an escape hatch to avoid learning the borrowing rules. It's a targeted solution for specific patterns — graph structures, shared state, observer patterns — where aliasing and mutation genuinely must coexist. If you find yourself wrapping everything in `RefCell`, you probably need to redesign your data flow.

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

## Non-Lexical Lifetimes (NLL) — The Borrow Checker Got Smarter

If you read Rust blog posts or Stack Overflow answers from 2015-2017, you'll see a **lot** of complaints about "fighting the borrow checker." Many of those complaints are no longer relevant, thanks to **Non-Lexical Lifetimes (NLL)**.

### The Old World: Lexical Lifetimes (pre-2018)

Before Rust 2018, borrows lasted until the **end of the enclosing scope** — the closing `}` brace. This was simple to implement but **overly conservative**:

```rust
// Pre-2018: This was REJECTED even though it's perfectly safe
fn main() {
    let mut v = Vec::new();
    let r = &v;           // Immutable borrow starts
    println!("{:?}", r);  // Last actual use of r
    // ... r is never used again ...
    v.push(1);            // ❌ OLD COMPILER: "v is still borrowed!"
}                          //    Borrow lasted until this brace
```

The borrow on `r` was **clearly finished** after `println!`, but the old compiler kept it alive until `}` because that's where `r`'s lexical scope ended.

### The Fix: Borrows End at Last Use

NLL, stabilized in the **Rust 2018 edition** (implemented 2018-2019), changed the rule: a borrow ends at its **last use point**, not at the closing brace.

```
OLD (Lexical):                    NEW (NLL):
  let r = &v;                       let r = &v;
  println!("{:?}", r);              println!("{:?}", r);
  │                                  // r's borrow ENDS HERE ✓
  │  ← borrow still alive           v.push(1);  // ✅ OK!
  │
  v.push(1);  // ❌ REJECTED
  }  ← borrow ends here
```

### How NLL Works Under the Hood

NLL doesn't just use simple heuristics — it performs **precise dataflow analysis** on Rust's MIR (Mid-level Intermediate Representation):

```
Source Code          HIR              MIR              LLVM IR
  (text)    →  (syntax tree)  →  (control-flow   →  (machine
                                  graph + basic       code)
                                  blocks)
                                      ↑
                                NLL analyzes HERE:
                                - Builds CFG
                                - Computes liveness
                                  intervals for
                                  each borrow
                                - Checks no
                                  conflicting
                                  borrows overlap
```

The compiler builds a **control-flow graph** of your function and computes **liveness intervals** — the precise range of program points where each borrow is "live" (i.e., might still be used). Two borrows conflict only if their liveness intervals **actually overlap**. This is far more precise than "same lexical scope."

### Polonius: The Next Generation

**Polonius** is the next-generation borrow checker currently being developed (as of 2025-2026, available experimentally with `-Z polonius`). It computes even more precise lifetime information using a **Datalog-based** analysis. Programs that NLL still conservatively rejects — particularly those involving conditional borrows and complex control flow — will compile under Polonius:

```rust
// NLL rejects this. Polonius accepts it.
fn get_or_insert(map: &mut HashMap<String, String>, key: &str) -> &String {
    if let Some(val) = map.get(key) {
        return val;  // Borrow from map escapes via return
    }
    map.insert(key.to_string(), "default".to_string());
    &map[key]
}
```

### The Practical Impact

NLL was one of the **single biggest quality-of-life improvements** in Rust's history. If you encounter old tutorials or blog posts that show "the borrow checker is too strict" examples — try them on a modern Rust compiler. Most of them just work now.

| Era | Borrow Scope | Common Complaint |
|-----|-------------|------------------|
| Pre-2018 (Lexical) | Until closing `}` | "I have to add ugly extra scopes everywhere" |
| 2018+ (NLL) | Until last use | "It mostly just works" |
| Future (Polonius) | Precise per-path analysis | "Even complex patterns compile" |

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

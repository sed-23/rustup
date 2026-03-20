# The Three Rules of Ownership 📜

> **Rust's entire memory safety guarantee comes from three simple rules. Learn these, and you'll understand why Rust code works the way it does.**

---

## Table of Contents

- [The Three Rules](#the-three-rules)
- [Rule 1: Each Value Has Exactly One Owner](#rule-1-each-value-has-exactly-one-owner)
  - [What Is an "Owner"?](#what-is-an-owner)
  - [Visualizing Ownership](#visualizing-ownership)
  - [Ownership of Different Types](#ownership-of-different-types)
- [Rule 2: There Can Only Be One Owner at a Time](#rule-2-there-can-only-be-one-owner-at-a-time)
  - [Why Only One Owner?](#why-only-one-owner)
  - [What Happens When You Assign?](#what-happens-when-you-assign)
- [Rule 3: When the Owner Goes Out of Scope, the Value Is Dropped](#rule-3-when-the-owner-goes-out-of-scope-the-value-is-dropped)
  - [What Is Scope?](#what-is-scope)
  - [The drop Function](#the-drop-function)
  - [Nested Scopes](#nested-scopes)
  - [RAII: Resource Acquisition Is Initialization](#raii-resource-acquisition-is-initialization)
- [Ownership in Action: Tracing Through Code](#ownership-in-action-tracing-through-code)
- [Why These Rules Prevent Bugs](#why-these-rules-prevent-bugs)
  - [No Double Free](#no-double-free)
  - [No Use-After-Free](#no-use-after-free)
  - [No Memory Leaks (mostly)](#no-memory-leaks-mostly)
  - [No Dangling Pointers](#no-dangling-pointers)
- [Ownership and Functions](#ownership-and-functions)
  - [Passing Values to Functions](#passing-values-to-functions)
  - [Returning Values from Functions](#returning-values-from-functions)
  - [Giving and Taking Back](#giving-and-taking-back)
- [The Tedium Problem (Preview of Borrowing)](#the-tedium-problem-preview-of-borrowing)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Three Rules

Here they are — commit them to memory:

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║  Rule 1: Each value in Rust has a variable that's called     ║
║          its OWNER.                                          ║
║                                                              ║
║  Rule 2: There can only be ONE owner at a time.             ║
║                                                              ║
║  Rule 3: When the owner goes out of scope, the value        ║
║          is DROPPED (freed/destroyed).                       ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

That's it. Three rules. Everything about Rust's memory model — every borrow checker error, every lifetime annotation, every `move` — traces back to these three rules.

---

## Rule 1: Each Value Has Exactly One Owner

### What Is an "Owner"?

The **owner** is the variable that is responsible for a value. When we say a variable "owns" a value, we mean:

- The variable **controls the memory** where the value lives
- The variable **decides when** the value gets cleaned up
- **Nobody else** can clean up that memory

```rust
fn main() {
    let s = String::from("hello");  // s is the OWNER of this String
    //  ↑         ↑
    //  owner     value (the String "hello")
    
    let x = 42;  // x is the OWNER of this i32 value
    //  ↑    ↑
    //  owner value (the number 42)
}
```

### Visualizing Ownership

Think of it like a **name tag** attached to a piece of data:

```
  ┌──────────┐     ┌───────────────────┐
  │  s       │────→│ String: "hello"    │
  │ (owner)  │     │ (value on heap)    │
  └──────────┘     └───────────────────┘

  ┌──────────┐     ┌───────────────────┐
  │  x       │────→│ i32: 42            │
  │ (owner)  │     │ (value on stack)   │
  └──────────┘     └───────────────────┘
```

Every value has exactly ONE name tag pointing to it. No value exists without an owner, and no value has two owners.

### Ownership of Different Types

```rust
fn main() {
    // Simple types — owner is the variable
    let age: i32 = 30;           // age owns 30
    let pi: f64 = 3.14159;      // pi owns 3.14159
    let flag: bool = true;      // flag owns true
    
    // Compound types — owner is the variable
    let point: (i32, i32) = (10, 20);     // point owns the tuple
    let arr: [i32; 3] = [1, 2, 3];        // arr owns the array
    
    // Heap types — owner is the variable
    let name: String = String::from("Alice");  // name owns the String
    let nums: Vec<i32> = vec![1, 2, 3];        // nums owns the Vec
    
    // Each owner is responsible for its value's memory
}
```

---

## Rule 2: There Can Only Be One Owner at a Time

### Why Only One Owner?

Imagine two people own the same house and both decide to demolish it independently. The first person demolishes it — fine. The second person tries to demolish it too — but it's already gone! This is a **double free** bug, and it causes crashes and security vulnerabilities.

Rust prevents this by allowing **only one owner at a time**.

```
❌ Not allowed in Rust:

  ┌──────────┐     ┌───────────────────┐     ┌──────────┐
  │  s1      │────→│ String: "hello"    │←───│  s2      │
  │ (owner)  │     │                    │     │ (owner)  │
  └──────────┘     └───────────────────┘     └──────────┘

  Who frees "hello"? s1? s2? Both? → BUG!

✅ Allowed in Rust:

  ┌──────────┐     ┌───────────────────┐
  │  s1      │────→│ String: "hello"    │
  │ (owner)  │     │                    │
  └──────────┘     └───────────────────┘

  Only s1 can free "hello". No ambiguity.
```

### What Happens When You Assign?

When you assign a heap-allocated value to another variable, ownership **moves**:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // Ownership MOVES from s1 to s2
    
    // s1 is now INVALID — using it is a compile error
    // println!("{}", s1);  // ❌ ERROR: value borrowed after move
    
    println!("{}", s2);  // ✅ s2 is the owner now
}
```

```
Before assignment:              After assignment:

  s1 ────→ "hello" (heap)        s1 ── ✗ (invalid)
                                  s2 ────→ "hello" (heap)
```

The data on the heap **doesn't move** — only the ownership (the pointer) moves from `s1` to `s2`. After the move, `s1` is no longer valid.

> We'll explore move semantics in full detail in the next tutorial.

---

## Rule 3: When the Owner Goes Out of Scope, the Value Is Dropped

### What Is Scope?

A **scope** is the region of code where a variable is valid. In Rust, scope is typically defined by curly braces `{ }`:

```rust
fn main() {
    // x is not yet valid — hasn't been declared
    
    {
        let x = 42;  // x enters scope — x is valid from here
        println!("{}", x);  // ✅ x is valid
    }  // ← x goes out of scope here — x is dropped (freed)
    
    // println!("{}", x);  // ❌ ERROR: x is not in scope
}
```

### Scope Visualization

```rust
fn main() {                         // ─── main scope begins
    let a = String::from("hello");  // a enters scope
    
    {                                // ─── inner scope begins
        let b = String::from("world");  // b enters scope
        println!("{} {}", a, b);    // ✅ both valid
    }                                // ─── inner scope ends → b is DROPPED
    
    println!("{}", a);              // ✅ a is still valid
    // println!("{}", b);           // ❌ b no longer exists
    
}                                    // ─── main scope ends → a is DROPPED
```

```
Timeline:

        main starts
           │
    ┌──────┤ a created ────────────────────────────── a dropped ─┐
    │      │                                                     │
    │      │ ┌── inner scope ──┐                                 │
    │      │ │ b created ──── b dropped                          │
    │      │ └─────────────────┘                                 │
    │      │                                                     │
    └──────┘                                                     │
           │                                                     │
        main ends ───────────────────────────────────────────────┘
```

### The `drop` Function

When a value goes out of scope, Rust automatically calls a special function called `drop`. For `String`, `drop` frees the heap memory. For a `File`, `drop` closes the file handle. This is automatic — you never call `drop` yourself in normal code.

```rust
fn main() {
    let s = String::from("hello");
    println!("{}", s);
}   // ← Rust automatically calls drop(s) here
    //   This frees the heap memory for "hello"
```

What `drop` does depends on the type:

| Type | What `drop` Does |
|------|-----------------|
| `String` | Frees heap memory containing the text |
| `Vec<T>` | Drops each element, then frees heap memory |
| `File` | Closes the file handle |
| `MutexGuard` | Releases the lock |
| `TcpStream` | Closes the network connection |
| `i32`, `f64`, `bool` | Nothing (stack memory is reclaimed automatically) |

### Nested Scopes

Values are dropped in **reverse order** of creation (like a stack):

```rust
fn main() {
    let first = String::from("first");    // Created 1st
    let second = String::from("second");  // Created 2nd
    let third = String::from("third");    // Created 3rd
    
    println!("{} {} {}", first, second, third);
}
// Dropped in reverse order:
// 1. third is dropped  (created last, freed first)
// 2. second is dropped
// 3. first is dropped  (created first, freed last)
```

This reverse order matters when values reference each other (we'll see this with lifetimes later).

### RAII: Resource Acquisition Is Initialization

Rust uses a pattern called **RAII** (borrowed from C++). The idea:

- **Acquiring** a resource (memory, file, lock, connection) = creating a value
- **Releasing** a resource = dropping the value (when it goes out of scope)

This means resources are **always** cleaned up, even if your code panics:

```rust
fn read_file() -> Result<String, std::io::Error> {
    let file = std::fs::File::open("data.txt")?;
    // If an error occurs on the next line, file is STILL dropped
    // (the file handle is closed automatically)
    let contents = std::io::read_to_string(file)?;
    Ok(contents)
}   // file goes out of scope and is closed
```

In C, if you forget to call `fclose()`, the file stays open until the program exits. In Rust, it's impossible to forget — `drop` handles it.

---

## Ownership in Action: Tracing Through Code

Let's trace step by step through a program:

```rust
fn main() {
    let s1 = String::from("hello");  // Step 1
    let s2 = String::from("world");  // Step 2
    let s3 = s1;                      // Step 3
    println!("{}", s3);               // Step 4
    println!("{}", s2);               // Step 5
}                                      // Step 6
```

```
Step 1: s1 = String::from("hello")
┌──────┐      Heap
│ s1 ──│────→ "hello"
└──────┘
Owners: s1 → "hello"

Step 2: s2 = String::from("world")
┌──────┐      Heap
│ s1 ──│────→ "hello"
│ s2 ──│────→ "world"
└──────┘
Owners: s1 → "hello", s2 → "world"

Step 3: s3 = s1  (MOVE — s1 becomes invalid)
┌──────┐      Heap
│ s1 ✗ │      "hello" ←────│── s3
│ s2 ──│────→ "world"      │
│ s3 ──│──────────────────→─┘
└──────┘
Owners: s3 → "hello", s2 → "world"
Invalid: s1

Step 4: println!("{}", s3)  → prints "hello" ✅

Step 5: println!("{}", s2)  → prints "world" ✅

Step 6: End of main — cleanup
  - s3 is dropped → "hello" freed from heap
  - s2 is dropped → "world" freed from heap
  - s1 is invalid, nothing to drop
```

---

## Why These Rules Prevent Bugs

### No Double Free

In C/C++, if two pointers point to the same memory and both try to free it:

```c
// C code — DOUBLE FREE BUG
char *s1 = malloc(6);
strcpy(s1, "hello");
char *s2 = s1;        // Both point to same memory!
free(s1);             // Free once — OK
free(s2);             // Free again — 💥 CRASH OR SECURITY HOLE
```

In Rust, the single-owner rule makes this impossible:

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 is MOVED — s1 no longer owns the data
// Only s2 will free the memory when it goes out of scope
// Double free is IMPOSSIBLE
```

### No Use-After-Free

```c
// C code — USE AFTER FREE BUG
char *s = malloc(6);
strcpy(s, "hello");
free(s);
printf("%s\n", s);  // 💥 s points to freed memory — undefined behavior!
```

In Rust:

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 is moved
// println!("{}", s1);  // ❌ Compile error! s1 is invalid
// The compiler catches this AT COMPILE TIME — before your program runs
```

### No Memory Leaks (mostly)

Because every owned value is dropped when it goes out of scope, memory is **always** freed:

```rust
fn create_lots_of_strings() {
    for i in 0..1_000_000 {
        let s = String::from("data");
        // s is dropped at the end of each iteration
        // Memory is freed immediately — no leak!
    }
    // After this loop, 0 bytes are leaked
}
```

> **Note:** Rust doesn't guarantee zero leaks. You can intentionally leak with `std::mem::forget` or `Box::leak`, and reference cycles with `Rc` can leak. But accidental leaks are extremely rare.

### No Dangling Pointers

A dangling pointer points to memory that has been freed. Rust's borrow checker prevents this:

```rust
// fn dangle() -> &String {
//     let s = String::from("hello");
//     &s  // ❌ ERROR: s is about to be dropped, can't return a reference to it!
// }

fn no_dangle() -> String {
    let s = String::from("hello");
    s  // ✅ Move the String out — the caller becomes the new owner
}
```

---

## Ownership and Functions

### Passing Values to Functions

When you pass a value to a function, ownership **moves** to the function's parameter:

```rust
fn take_ownership(s: String) {
    println!("I own: {}", s);
}   // s is dropped here — the String is freed

fn main() {
    let greeting = String::from("hello");
    take_ownership(greeting);   // greeting's ownership moves to s
    
    // println!("{}", greeting);  // ❌ ERROR! greeting is no longer valid
}
```

```
Before function call:         During function call:         After function returns:

main:                         take_ownership:                main:
┌──────────┐                  ┌──────────┐                  ┌──────────┐
│ greeting ─┼─→ "hello"      │ s ────────┼─→ "hello"      │ greeting ✗│
└──────────┘                  └──────────┘                  └──────────┘
                              main:                          Heap: "hello" FREED
                              ┌──────────┐
                              │ greeting ✗│
                              └──────────┘
```

### Stack Types Are Different (Copy)

For types that implement the `Copy` trait (integers, floats, booleans, characters), the value is **copied** instead of moved:

```rust
fn take_number(n: i32) {
    println!("I have: {}", n);
}

fn main() {
    let x = 42;
    take_number(x);       // x is COPIED, not moved
    println!("x = {}", x);  // ✅ x is still valid!
}
```

Why? Because copying a small stack value (4 bytes for i32) is trivial — there's no heap memory to worry about. We'll explore this in detail in the Clone & Copy tutorial.

### Returning Values from Functions

Functions can **transfer ownership** back to the caller via return values:

```rust
fn create_greeting() -> String {
    let s = String::from("Hello, world!");
    s  // Ownership moves to the caller
}   // s is NOT dropped here — ownership was transferred

fn main() {
    let greeting = create_greeting();  // greeting is now the owner
    println!("{}", greeting);          // ✅ valid
}   // greeting is dropped here — "Hello, world!" is freed
```

### Giving and Taking Back

A common (but tedious) pattern before we learn about borrowing:

```rust
fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)  // Give back ownership AND the result
}

fn main() {
    let s1 = String::from("hello");
    
    let (s2, len) = calculate_length(s1);
    // s1 is moved into the function
    // The function gives it back as s2
    
    println!("The length of '{}' is {}", s2, len);
}
```

This works but it's **ugly and tedious**. Imagine if every function had to return every value it received! This is exactly why Rust has **borrowing** (which we'll learn soon).

---

## The Tedium Problem (Preview of Borrowing)

The ownership rules create a problem — passing values to functions moves them, so you lose access. The workaround (returning the value back) is tedious:

```rust
// ❌ Tedious: have to return everything back
fn print_length(s: String) -> String {
    println!("Length: {}", s.len());
    s  // Give it back
}

fn print_uppercase(s: String) -> String {
    println!("Upper: {}", s.to_uppercase());
    s  // Give it back again
}

fn main() {
    let s = String::from("hello");
    let s = print_length(s);       // Take and give back
    let s = print_uppercase(s);    // Take and give back again
    println!("Still have: {}", s);
}
```

The solution is **borrowing** — letting a function *use* a value without *taking ownership*:

```rust
// ✅ With borrowing (preview — covered in tutorial 5)
fn print_length(s: &String) {
    println!("Length: {}", s.len());
}   // s is a reference — no ownership to give back

fn main() {
    let s = String::from("hello");
    print_length(&s);  // Lend s temporarily
    println!("Still mine: {}", s);  // ✅ s is still valid
}
```

Don't worry about the `&` syntax yet — we'll cover it thoroughly in the References & Borrowing tutorial.

---

## Common Mistakes

### Mistake 1: Using a Value After Moving It

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    
    // println!("{}", s1);  // ❌ ERROR: value used here after move
    println!("{}", s2);     // ✅ s2 is the owner now
}
```

**Fix:** Use `s2` instead, or use `.clone()` if you need both to have the data (next tutorial).

### Mistake 2: Using a Value After Passing It to a Function

```rust
fn consume(s: String) {
    println!("{}", s);
}

fn main() {
    let my_string = String::from("important");
    consume(my_string);
    
    // println!("{}", my_string);  // ❌ ERROR: my_string was moved
}
```

**Fix:** Either return the value from the function, clone before passing, or use a reference (`&`).

### Mistake 3: Expecting Assignment to Copy Heap Data

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // This is a MOVE, not a copy!
    
    // WRONG assumption: s1 and s2 are independent copies
    // CORRECT: s1 is invalid, s2 is the sole owner
}
```

### Mistake 4: Confusing Ownership with Scope

```rust
fn main() {
    let s = String::from("hello");
    
    {
        let r = &s;  // r borrows s — s still owns the data
        println!("{}", r);
    }   // r goes out of scope, but s is STILL valid
    
    println!("{}", s);  // ✅ s is still the owner
}
```

A reference going out of scope doesn't affect the owner. Only when the **owner** goes out of scope is the value dropped.

---

## Exercises

### Exercise 1: Tracing Ownership

For each line, state who owns the String:

```rust
fn main() {
    let a = String::from("Rust");     // Line 1
    let b = a;                         // Line 2
    let c = b;                         // Line 3
    println!("{}", c);                 // Line 4
}                                      // Line 5
```

<details>
<summary>Answer</summary>

| Line | Owner of "Rust" | Invalid variables |
|------|-----------------|-------------------|
| 1 | `a` | (none) |
| 2 | `b` | `a` |
| 3 | `c` | `a`, `b` |
| 4 | `c` (printed) | `a`, `b` |
| 5 | (dropped) | `a`, `b`, `c` |

</details>

### Exercise 2: Will It Compile?

Predict whether each program compiles, and if not, which line causes the error:

**Program A:**
```rust
fn main() {
    let s = String::from("hello");
    let t = s;
    println!("{}", t);
}
```

**Program B:**
```rust
fn main() {
    let s = String::from("hello");
    let t = s;
    println!("{} {}", s, t);
}
```

**Program C:**
```rust
fn main() {
    let x = 42;
    let y = x;
    println!("{} {}", x, y);
}
```

<details>
<summary>Answer</summary>

- **A:** ✅ Compiles. `s` is moved to `t`, only `t` is used.
- **B:** ❌ Error on `println!` line. `s` was moved to `t`, then `s` is used.
- **C:** ✅ Compiles. `i32` is `Copy`, so `x` is copied, both are valid.

</details>

### Exercise 3: Fix the Code

This code doesn't compile. Fix it **without** using references (we haven't learned those yet). You can use clone or restructure the code:

```rust
fn print_string(s: String) {
    println!("{}", s);
}

fn main() {
    let message = String::from("Hello!");
    print_string(message);
    print_string(message);  // ❌ Error: message was moved
}
```

<details>
<summary>Answer</summary>

Option 1: Clone before the first call:
```rust
fn main() {
    let message = String::from("Hello!");
    print_string(message.clone());
    print_string(message);
}
```

Option 2: Return ownership:
```rust
fn print_string(s: String) -> String {
    println!("{}", s);
    s
}

fn main() {
    let message = String::from("Hello!");
    let message = print_string(message);
    print_string(message);
}
```

</details>

### Exercise 4: Scope and Drop Order

What order are values dropped in this code?

```rust
fn main() {
    let a = String::from("A");
    let b = String::from("B");
    {
        let c = String::from("C");
        let d = String::from("D");
    }
    let e = String::from("E");
}
```

<details>
<summary>Answer</summary>

Drop order:
1. `d` (inner scope ends — last created in inner scope)
2. `c` (inner scope ends — first created in inner scope)
3. `e` (main scope ends — last created in main)
4. `b` (main scope ends)
5. `a` (main scope ends — first created in main)

Remember: values are dropped in **reverse order** of creation, and inner scopes are cleaned up first.

</details>

---

## Summary

### The Three Rules

| Rule | Statement | Effect |
|------|-----------|--------|
| **1** | Each value has exactly one owner | Every piece of data belongs to one variable |
| **2** | There can only be one owner at a time | Assigning moves ownership |
| **3** | When the owner goes out of scope, the value is dropped | Automatic cleanup at predictable points |

### Key Concepts

| Concept | Meaning |
|---------|---------|
| **Owner** | The variable responsible for a value |
| **Move** | Transfer of ownership from one variable to another |
| **Drop** | Automatic cleanup when a value's owner goes out of scope |
| **Scope** | The region of code where a variable is valid (`{ }`) |
| **RAII** | Resources are acquired on creation and released on drop |

### What Ownership Prevents

| Bug | How Ownership Prevents It |
|-----|--------------------------|
| Double free | Only one owner → only one drop |
| Use-after-free | Moved values are invalid → compile error if used |
| Memory leaks | Owner is always dropped → memory always freed |
| Dangling pointers | Can't return references to local values |

### Key Takeaways

1. **Every value has exactly one owner** — the variable that controls it
2. **Assignment moves ownership** for heap types (String, Vec, etc.)
3. **When the owner goes out of scope, the value is dropped** — no GC, no manual free
4. **Moves make the source invalid** — using it is a compile error
5. **Simple types (i32, bool, etc.) are copied, not moved** — they implement `Copy`
6. **Functions consume ownership** when they take values by value
7. **This is Rust's core innovation** — memory safety without runtime cost

---

## What's Next?

We mentioned that assigning a heap value "moves" it. But what exactly happens in memory? And what about stack types that get copied instead?

**Next Tutorial:** [Move Semantics →](./03-move-semantics.md)

---

<p align="center">
  <i>Tutorial 2 of 8 — Stage 3: Ownership</i>
</p>

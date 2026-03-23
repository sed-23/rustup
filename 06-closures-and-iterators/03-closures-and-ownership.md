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

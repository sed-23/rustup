# Control Flow — Loops 🔄

> **Loops let your program repeat actions. Rust has three loop types: `loop`, `while`, and `for` — each for a different scenario.**

---

## Table of Contents

- [Why Loops?](#why-loops)
- [The Three Kinds of Loops](#the-three-kinds-of-loops)
- [loop — Infinite Loop](#loop--infinite-loop)
  - [Breaking Out with break](#breaking-out-with-break)
  - [Returning Values from loop](#returning-values-from-loop)
  - [Skipping Iterations with continue](#skipping-iterations-with-continue)
- [while — Conditional Loop](#while--conditional-loop)
  - [while with a Counter](#while-with-a-counter)
  - [while let — Looping on Patterns](#while-let--looping-on-patterns)
- [for — Iterating Over Collections](#for--iterating-over-collections)
  - [Iterating Over Ranges](#iterating-over-ranges)
  - [Iterating Over Arrays and Slices](#iterating-over-arrays-and-slices)
  - [Iterating with Index (enumerate)](#iterating-with-index-enumerate)
  - [Iterating Over Characters in a String](#iterating-over-characters-in-a-string)
  - [Iterating in Reverse](#iterating-in-reverse)
- [Loop Labels — Nested Loop Control](#loop-labels--nested-loop-control)
- [Comparison: loop vs while vs for](#comparison-loop-vs-while-vs-for)
- [Performance Considerations](#performance-considerations)
- [Common Patterns](#common-patterns)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Why Loops?

Without loops, you'd have to write repetitive code:

```rust
// ❌ Without loops — repetitive and fragile
println!("1");
println!("2");
println!("3");
println!("4");
println!("5");
// What if you need 1000 lines? 😱

// ✅ With a loop — concise and flexible
for i in 1..=5 {
    println!("{}", i);
}
// Change 5 to 1000 and it just works
```

Loops let you:
1. **Repeat code** a specific number of times
2. **Process collections** (arrays, vectors, etc.)
3. **Wait for conditions** (user input, network response)
4. **Run indefinitely** (game loop, server)

---

## The Three Kinds of Loops

| Loop Type | When to Use | Example |
|-----------|------------|---------|
| `loop` | Repeat forever (until `break`) | Game loops, servers, retry logic |
| `while` | Repeat while a condition is true | Reading until EOF, countdown |
| `for` | Iterate over a sequence | Arrays, ranges, collections |

```
loop:   "Keep going until I say stop"
while:  "Keep going while this is true"
for:    "Do this for each item in the sequence"
```

### The History of Loops in Computing

Loops are one of the fundamental reasons computers are useful — the ability to repeat instructions is what separates a computer from a calculator.

**Machine code (1940s):** The earliest "loop" was a **branch instruction** that jumped backward in the program. The programmer manually calculated the memory address to jump to. Getting it wrong meant the program ran forever or crashed.

**Fortran (1957):** Introduced the `DO` loop — the ancestor of all `for` loops:
```fortran
DO 10 I = 1, 100
    X = X + I
10 CONTINUE
```
The number `10` is a line label — the loop body extends from the `DO` to the `CONTINUE` labeled `10`.

**C (1972):** Introduced the three-part `for` loop that became ubiquitous:
```c
for (int i = 0; i < 100; i++) {
    // The most common loop in programming history
}
```

**Python (1991):** Made `for` about iteration, not counting:
```python
for item in collection:  # Iterate over items directly
    process(item)
```

**Rust (2015):** Follows Python's philosophy (iterator-based `for`) but adds:
- `loop` — a dedicated infinite loop (safer than `while (true)`)
- `while let` — pattern-matching loops (from ML tradition)
- `break` with values — loops as expressions (unique to Rust!)
- Loop labels — nested loop control

> **Why Three Loop Types?** Many languages have only `for` and `while` (C, Python, Java). Rust added `loop` as a separate construct because `while true { }` is a pattern — the compiler can reason about `loop` more effectively. It knows the loop ONLY exits via `break`, which enables the "return value from loop" feature and better optimization.

---

## `loop` — Infinite Loop

The simplest loop — runs forever until you explicitly `break`:

```rust
fn main() {
    let mut count = 0;
    
    loop {
        count += 1;
        println!("Count: {}", count);
        
        if count >= 5 {
            break;  // Exit the loop
        }
    }
    
    println!("Loop ended at count {}", count);
}
```

Output:
```
Count: 1
Count: 2
Count: 3
Count: 4
Count: 5
Loop ended at count 5
```

### How `loop` Works in Memory

```
Execution flow:

     ┌──→ Check body
     │         │
     │    Execute code
     │         │
     │    break? ──Yes──→ Exit loop
     │         │
     │        No
     │         │
     └─────────┘
```

### Without `break`, It Runs Forever

```rust
fn main() {
    loop {
        println!("This prints forever!");
        // No break — this is an infinite loop
        // Press Ctrl+C to stop the program
    }
}
```

### Real-World Use: Server/Game Loop

```rust
fn main() {
    println!("Game started!");
    let mut frame = 0;
    
    loop {
        frame += 1;
        // process_input();
        // update_game_state();
        // render_frame();
        
        println!("Frame {}", frame);
        
        if frame >= 3 {
            println!("Game over!");
            break;
        }
    }
}
```

### Breaking Out with `break`

`break` immediately exits the innermost loop:

```rust
fn main() {
    let mut sum = 0;
    let mut i = 1;
    
    loop {
        sum += i;
        
        if sum > 100 {
            println!("Sum exceeded 100 at i={}", i);
            break;
        }
        
        i += 1;
    }
    
    println!("Final sum: {}", sum);
    // Sum exceeded 100 at i=14
    // Final sum: 105
}
```

### Returning Values from `loop`

`loop` is an expression — `break` can return a value!

```rust
fn main() {
    let mut counter = 0;
    
    let result = loop {
        counter += 1;
        
        if counter == 10 {
            break counter * 2;  // ← Returns 20 from the loop
        }
    };
    
    println!("Result: {}", result);  // Result: 20
}
```

This is unique to Rust's `loop` — most languages can't return values from loops.

### Real-World Use: Retry Until Success

```rust
fn main() {
    let mut attempts = 0;
    
    let connection = loop {
        attempts += 1;
        println!("Attempt {}...", attempts);
        
        // Simulate: succeed on 3rd attempt
        if attempts == 3 {
            break "Connected!";
        }
        
        println!("Failed, retrying...");
    };
    
    println!("{} (after {} attempts)", connection, attempts);
    // Attempt 1...
    // Failed, retrying...
    // Attempt 2...
    // Failed, retrying...
    // Attempt 3...
    // Connected! (after 3 attempts)
}
```

### Deep Dive: Loops as Expressions — Rust's Unique Feature

The ability to return values from `loop` with `break value` is one of Rust's unique features. No other mainstream systems programming language has this.

**Why is this useful?** It eliminates a common pattern of initializing a "result" variable before the loop:

```rust
// Without loop-as-expression (traditional approach):
let mut result = String::new();  // Must declare outside the loop
loop {
    let line = read_line();
    if line == "quit" {
        result = format!("Received {} lines", count);
        break;
    }
}
// result is used here

// With loop-as-expression (Rust way):
let result = loop {  // result is initialized from loop's return value
    let line = read_line();
    if line == "quit" {
        break format!("Received {} lines", count);  // 🎯 Direct
    }
};
```

The second version is better because:
1. `result` doesn't need to be `mut`
2. `result` is guaranteed to be initialized (the compiler proves all `break` paths provide a value)
3. The intent is clearer — "this loop exists to compute a result"

This is another example of Rust's **expression-oriented** design (inherited from ML/Haskell), where constructs produce values rather than just executing statements.

### Skipping Iterations with `continue`

`continue` skips the rest of the current iteration and jumps back to the top:

```rust
fn main() {
    let mut i = 0;
    
    loop {
        i += 1;
        
        if i > 10 {
            break;
        }
        
        if i % 2 == 0 {
            continue;  // Skip even numbers
        }
        
        println!("{}", i);  // Only prints odd numbers
    }
}
```

Output:
```
1
3
5
7
9
```

### Flow with `continue`

```
     ┌──→ Increment i
     │         │
     │    i > 10? ──Yes──→ break (exit)
     │         │
     │        No
     │         │
     │    i even? ──Yes──→ continue (skip to top) ──┐
     │         │                                     │
     │        No                                     │
     │         │                                     │
     │    println!                                   │
     │         │                                     │
     └─────────┘  ←────────────────────────────────┘
```

---

## `while` — Conditional Loop

`while` repeats as long as a condition is `true`:

```rust
fn main() {
    let mut countdown = 5;
    
    while countdown > 0 {
        println!("{}...", countdown);
        countdown -= 1;
    }
    
    println!("Liftoff! 🚀");
}
```

Output:
```
5...
4...
3...
2...
1...
Liftoff! 🚀
```

### How `while` Works

```
     ┌──→ Check condition
     │         │
     │    True?  ──No──→ Exit loop
     │         │
     │        Yes
     │         │
     │    Execute body
     │         │
     └─────────┘
```

**Key difference from `loop`:** The condition is checked **before** each iteration. If the condition is `false` from the start, the body never executes:

```rust
fn main() {
    let mut x = 10;
    
    while x < 5 {
        println!("This never prints!");
        x += 1;
    }
    // x starts at 10, which is NOT < 5, so the loop body is skipped entirely
}
```

### `while` with a Counter

```rust
fn main() {
    let mut i = 0;
    
    while i < 5 {
        println!("i = {}", i);
        i += 1;
    }
    // i = 0, 1, 2, 3, 4
}
```

### Iterating Over an Array with `while`

```rust
fn main() {
    let numbers = [10, 20, 30, 40, 50];
    let mut index = 0;
    
    while index < numbers.len() {
        println!("numbers[{}] = {}", index, numbers[index]);
        index += 1;
    }
}
```

> **Note:** The `for` loop is better for this — see the next section.

### `while let` — Looping on Patterns

Similar to `if let`, but loops as long as the pattern matches:

```rust
fn main() {
    let mut stack = vec![1, 2, 3, 4, 5];
    
    // pop() returns Option<T>: Some(value) or None
    while let Some(top) = stack.pop() {
        println!("Popped: {}", top);
    }
    // Popped: 5
    // Popped: 4
    // Popped: 3
    // Popped: 2
    // Popped: 1
    // (Loop ends when pop() returns None)
    
    println!("Stack is empty: {:?}", stack);
}
```

### `while let` is Equivalent to:

```rust
loop {
    match stack.pop() {
        Some(top) => println!("Popped: {}", top),
        None => break,
    }
}
```

But `while let` is much cleaner.

---

## `for` — Iterating Over Collections

The `for` loop is the **most common** loop in Rust. It iterates over anything that implements the `Iterator` trait:

```rust
fn main() {
    let fruits = ["apple", "banana", "cherry"];
    
    for fruit in fruits {
        println!("I like {}!", fruit);
    }
}
```

Output:
```
I like apple!
I like banana!
I like cherry!
```

### The Syntax

```
for variable in iterable {
    // body — runs once for each item
}
```

### Iterating Over Ranges

```rust
fn main() {
    // Exclusive range: 1, 2, 3, 4 (does NOT include 5)
    for i in 1..5 {
        print!("{} ", i);
    }
    println!();
    // Output: 1 2 3 4
    
    // Inclusive range: 1, 2, 3, 4, 5 (INCLUDES 5)
    for i in 1..=5 {
        print!("{} ", i);
    }
    println!();
    // Output: 1 2 3 4 5
    
    // Starting from 0
    for i in 0..3 {
        print!("{} ", i);
    }
    println!();
    // Output: 0 1 2
}
```

### Range Types

```
1..5    → [1, 2, 3, 4]         Exclusive range (excludes end)
1..=5   → [1, 2, 3, 4, 5]     Inclusive range (includes end)
0..n    → [0, 1, 2, ..., n-1]  Common for array indexing
```

### Iterating Over Arrays and Slices

There are three ways to iterate, each with different ownership implications:

```rust
fn main() {
    let numbers = [10, 20, 30, 40, 50];
    
    // Method 1: for item in array (moves/copies elements)
    println!("Method 1: consuming");
    for num in numbers {
        // num is i32 (copy — because i32 implements Copy)
        print!("{} ", num);
    }
    println!();
    
    // Method 2: for item in &array (borrows elements — most common)
    println!("Method 2: borrowing");
    for num in &numbers {
        // num is &i32 (reference to each element)
        print!("{} ", num);
    }
    println!();
    
    // Method 3: for item in &mut array (mutable borrow)
    let mut mutable_numbers = [10, 20, 30, 40, 50];
    println!("Method 3: mutable borrow");
    for num in &mut mutable_numbers {
        *num *= 2;  // Double each element
    }
    println!("Doubled: {:?}", mutable_numbers);
    // [20, 40, 60, 80, 100]
}
```

### Method Cheatsheet

| Syntax | Type of `item` | Ownership |
|--------|---------------|-----------|
| `for item in collection` | `T` | Takes ownership (or copies) |
| `for item in &collection` | `&T` | Borrows (read-only) |
| `for item in &mut collection` | `&mut T` | Borrows (mutable) |

### Iterating with Index (`enumerate`)

```rust
fn main() {
    let colors = ["red", "green", "blue", "yellow"];
    
    for (index, color) in colors.iter().enumerate() {
        println!("Color {}: {}", index, color);
    }
}
```

Output:
```
Color 0: red
Color 1: green
Color 2: blue
Color 3: yellow
```

How `enumerate` works:
```
colors.iter()     → ["red", "green", "blue", "yellow"]
    .enumerate()  → [(0, "red"), (1, "green"), (2, "blue"), (3, "yellow")]
```

### Iterating Over Characters in a String

```rust
fn main() {
    let message = "Hello, 🌍!";
    
    // Iterate over characters (Unicode-aware)
    for ch in message.chars() {
        println!("'{}'", ch);
    }
    // 'H', 'e', 'l', 'l', 'o', ',', ' ', '🌍', '!'
    
    // Iterate over bytes (raw bytes)
    for byte in message.bytes() {
        print!("{} ", byte);
    }
    println!();
    // 72 101 108 108 111 44 32 240 159 140 141 33
    // Notice: 🌍 = 4 bytes!
    
    // Iterate with character index
    for (i, ch) in message.char_indices() {
        println!("Index {}: '{}'", i, ch);
    }
}
```

### Iterating in Reverse

```rust
fn main() {
    // Reverse a range
    for i in (1..=5).rev() {
        print!("{} ", i);
    }
    println!();
    // Output: 5 4 3 2 1
    
    // Countdown
    for i in (0..=10).rev().step_by(2) {
        print!("{} ", i);
    }
    println!();
    // Output: 10 8 6 4 2 0
}
```

### Step By

```rust
fn main() {
    // Count by 2s
    for i in (0..=20).step_by(2) {
        print!("{} ", i);
    }
    println!();
    // 0 2 4 6 8 10 12 14 16 18 20
    
    // Count by 5s
    for i in (0..=50).step_by(5) {
        print!("{} ", i);
    }
    println!();
    // 0 5 10 15 20 25 30 35 40 45 50
}
```

### Why `for` is Better Than `while` for Collections

```rust
fn main() {
    let arr = [10, 20, 30, 40, 50];
    
    // ❌ while loop: verbose, error-prone
    // (can forget to increment, can go out of bounds)
    let mut i = 0;
    while i < arr.len() {
        println!("{}", arr[i]);
        i += 1;
    }
    
    // ✅ for loop: concise, safe, no off-by-one errors
    for val in &arr {
        println!("{}", val);
    }
}
```

The `for` loop:
- **Can't go out of bounds** — the iterator handles bounds
- **No index variable** to manage
- **Often faster** — the compiler can optimize better when it knows the loop pattern

---

## Loop Labels — Nested Loop Control

When you have nested loops, `break` and `continue` only affect the **innermost** loop. Use **labels** to target outer loops:

```rust
fn main() {
    // Label: 'outer (starts with a single quote)
    'outer: for i in 0..3 {
        'inner: for j in 0..3 {
            if i == 1 && j == 1 {
                println!("Breaking outer loop at i={}, j={}", i, j);
                break 'outer;  // ← Breaks the OUTER loop
            }
            println!("i={}, j={}", i, j);
        }
    }
    println!("Done!");
}
```

Output:
```
i=0, j=0
i=0, j=1
i=0, j=2
i=1, j=0
Breaking outer loop at i=1, j=1
Done!
```

### Without Label (Only Breaks Inner)

```rust
fn main() {
    for i in 0..3 {
        for j in 0..3 {
            if j == 1 {
                break;  // Only breaks the INNER loop
            }
            println!("i={}, j={}", i, j);
        }
    }
}
```

Output:
```
i=0, j=0
i=1, j=0
i=2, j=0
```

### Label Syntax

```
'label_name: for/while/loop {
    break 'label_name;     // Break the labeled loop
    continue 'label_name;  // Continue the labeled loop
}
```

Labels must start with a single quote `'` (like lifetimes — not a coincidence):

```rust
'outer: loop { ... }
'search: for item in list { ... }
'retry: loop { ... }
```

### Practical Example: Searching a 2D Grid

```rust
fn main() {
    let grid = [
        [1, 2, 3],
        [4, 5, 6],
        [7, 8, 9],
    ];
    
    let target = 5;
    let mut found_at = (0, 0);
    
    'search: for (row, row_data) in grid.iter().enumerate() {
        for (col, &value) in row_data.iter().enumerate() {
            if value == target {
                found_at = (row, col);
                break 'search;
            }
        }
    }
    
    println!("Found {} at row={}, col={}", target, found_at.0, found_at.1);
    // Found 5 at row=1, col=1
}
```

### Loop Values with Labels

You can return values from labeled loops too:

```rust
fn main() {
    let result = 'outer: loop {
        let mut count = 0;
        loop {
            count += 1;
            if count == 5 {
                break 'outer count * 10;  // Returns 50 from outer loop
            }
        }
    };
    
    println!("Result: {}", result);  // 50
}
```

---

## Comparison: `loop` vs `while` vs `for`

| Feature | `loop` | `while` | `for` |
|---------|--------|---------|-------|
| Condition checked | Never (infinite) | Before each iteration | N/A (iterates collection) |
| Use when | Unknown iterations, retry logic | Condition-based repetition | Iterating over data |
| Can return value? | ✅ Yes (`break value`) | ❌ No | ❌ No |
| Common use | Servers, game loops | Countdowns, input parsing | Collections, ranges |
| Off-by-one risk? | Low | Medium | **None** |
| Bounds checked? | Manual | Manual | **Automatic** |

### When to Use Which

```rust
// USE loop WHEN:
// - You don't know when to stop
// - You need to return a value from the loop
// - Retry/reconnect logic

loop {
    let result = try_connect();
    if result.is_ok() {
        break result;
    }
}

// USE while WHEN:
// - You have a condition to check
// - The condition may be false from the start

while !queue.is_empty() {
    let item = queue.pop();
    process(item);
}

// USE for WHEN:
// - Iterating over a known sequence
// - Doing something N times
// - Processing all items in a collection (95% of loops in real code)

for line in file.lines() {
    process_line(line);
}
```

---

## Performance Considerations

### For Loops and Iterator Optimization

Rust's `for` loops compile to the same machine code as manually indexed loops, thanks to **zero-cost abstractions**:

```rust
// These produce identical machine code:

// Version 1: for loop
let mut sum = 0;
for x in &numbers {
    sum += x;
}

// Version 2: manual indexing
let mut sum = 0;
let mut i = 0;
while i < numbers.len() {
    sum += numbers[i];
    i += 1;
}

// Version 3: iterator method (also identical!)
let sum: i32 = numbers.iter().sum();
```

The compiler optimizes all three to the same assembly. **There is no performance penalty for using `for` loops in Rust.**

### The while Loop Can Be Slower

Surprisingly, `while` with manual indexing can be **slower** than `for` because the compiler must insert bounds checks:

```rust
// This has a bounds check on EVERY iteration:
while i < arr.len() {
    let val = arr[i];  // Bounds check: is i < len? (runtime cost!)
    i += 1;
}

// This ELIMINATES bounds checks (compiler proves they're unnecessary):
for val in &arr {
    // No bounds check needed — the iterator guarantees valid access
}
```

> **Real-World Practice:** Always prefer `for` over `while` for iterating over collections. It's both safer AND faster.

### Deep Dive: How Rust's Iterator Optimization Works

Rust's `for` loops are fast because of how `Iterator` works under the hood. When you write:

```rust
for x in &numbers {
    sum += x;
}
```

The compiler desugars this to:
```rust
let mut iter = (&numbers).into_iter();
loop {
    match iter.next() {
        Some(x) => { sum += x; }
        None => break,
    }
}
```

This looks like it should be slower — there's a `match` on every iteration, checking for `None`. But LLVM is incredibly good at optimizing this away. Through a process called **inlining** and **dead code elimination**, the final machine code looks like:

```asm
; Simplified x86-64 assembly for summing an array:
loop_start:
    add eax, [rsi]        ; Add current element to sum
    add rsi, 4            ; Move pointer to next element
    cmp rsi, rdi          ; Have we reached the end?
    jne loop_start        ; If not, continue
```

No `Option`. No `match`. No function calls. Just raw pointer arithmetic — exactly what hand-written C would produce. This is what **zero-cost abstraction** means in practice: the high-level Rust code compiles to the same machine code as low-level pointer manipulation.

The key enablers are:
1. **Monomorphization** — generic code is compiled separately for each concrete type
2. **Inlining** — `next()` calls are replaced with their bodies
3. **LLVM optimization passes** — dead branches are removed, bounds checks are hoisted out of loops

> **Benchmark Evidence:** The official Rust documentation notes that iterator-based loops often match or exceed hand-optimized C code. In some cases, LLVM can auto-vectorize iterator chains (using SIMD instructions) more easily than manual loops, because the iterator structure makes data flow analysis easier for the optimizer.

### Loop Unrolling

For small, known-size loops, the compiler may **unroll** the loop — replacing it with straight-line code:

```rust
// This loop:
let arr = [1, 2, 3, 4];
let mut sum = 0;
for x in &arr {
    sum += x;
}

// May compile to:
// sum = 1 + 2 + 3 + 4;  (no loop at all!)
```

---

## Common Patterns

### Pattern 1: Accumulator

```rust
fn main() {
    let numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3];
    
    let mut sum = 0;
    let mut max = i32::MIN;
    let mut min = i32::MAX;
    
    for &num in &numbers {
        sum += num;
        if num > max { max = num; }
        if num < min { min = num; }
    }
    
    let avg = sum as f64 / numbers.len() as f64;
    
    println!("Sum: {}, Avg: {:.1}, Min: {}, Max: {}", sum, avg, min, max);
    // Sum: 39, Avg: 3.9, Min: 1, Max: 9
}
```

### Pattern 2: Counting Occurrences

```rust
fn main() {
    let text = "hello world";
    let target = 'l';
    
    let mut count = 0;
    for ch in text.chars() {
        if ch == target {
            count += 1;
        }
    }
    
    println!("'{}' appears {} times in '{}'", target, count, text);
    // 'l' appears 3 times in 'hello world'
}
```

### Pattern 3: Building a New Collection

```rust
fn main() {
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    let mut evens = Vec::new();
    for &num in &numbers {
        if num % 2 == 0 {
            evens.push(num);
        }
    }
    
    println!("Even numbers: {:?}", evens);
    // Even numbers: [2, 4, 6, 8, 10]
}
```

### Pattern 4: Nested Loops — Multiplication Table

```rust
fn main() {
    println!("Multiplication Table:");
    println!("    │  1   2   3   4   5   6   7   8   9  10");
    println!("────┼────────────────────────────────────────");
    
    for i in 1..=10 {
        print!("{:>3} │", i);
        for j in 1..=10 {
            print!("{:>3} ", i * j);
        }
        println!();
    }
}
```

### Pattern 5: Fibonacci Sequence

```rust
fn main() {
    let n = 20;
    let mut a: u64 = 0;
    let mut b: u64 = 1;
    
    print!("Fibonacci: {} {}", a, b);
    
    for _ in 2..n {
        let next = a + b;
        a = b;
        b = next;
        print!(" {}", b);
    }
    println!();
}
```

Note the `_` in `for _ in 2..n` — we don't need the index variable, so we discard it with `_`.

### Pattern 6: User Input Loop (Simulated)

```rust
fn main() {
    let inputs = ["hello", "", "world", "quit", "ignored"];
    let mut index = 0;
    
    loop {
        if index >= inputs.len() {
            break;
        }
        
        let input = inputs[index];
        index += 1;
        
        if input.is_empty() {
            println!("(skipping empty input)");
            continue;
        }
        
        if input == "quit" {
            println!("Goodbye!");
            break;
        }
        
        println!("You entered: {}", input);
    }
}
```

Output:
```
You entered: hello
(skipping empty input)
You entered: world
Goodbye!
```

---

## Common Mistakes

### Mistake 1: Infinite Loop Without `break`

```rust
// ❌ This runs forever — no way out
// loop {
//     println!("stuck!");
// }

// ✅ Always have a break condition
let mut count = 0;
loop {
    count += 1;
    if count >= 5 { break; }
}
```

### Mistake 2: Off-by-One in Range

```rust
// Want to print 1-5:
for i in 1..5 {
    print!("{} ", i);
}
// Output: 1 2 3 4  ← Missing 5!

// ✅ Use inclusive range
for i in 1..=5 {
    print!("{} ", i);
}
// Output: 1 2 3 4 5
```

### Mistake 3: Modifying a Collection While Iterating

```rust
// ❌ Cannot borrow as mutable while iterating
// let mut nums = vec![1, 2, 3, 4, 5];
// for num in &nums {
//     if *num % 2 == 0 {
//         nums.push(*num * 2);  // ❌ Can't modify during iteration
//     }
// }

// ✅ Collect items to add, then add them
let nums = vec![1, 2, 3, 4, 5];
let to_add: Vec<i32> = nums.iter()
    .filter(|&&n| n % 2 == 0)
    .map(|&n| n * 2)
    .collect();
let mut result = nums.clone();
result.extend(to_add);
```

### Mistake 4: Forgetting to Update the Counter in `while`

```rust
// ❌ Infinite loop — i never changes
// let mut i = 0;
// while i < 5 {
//     println!("{}", i);
//     // Oops, forgot i += 1;
// }

// ✅ Always update the counter
let mut i = 0;
while i < 5 {
    println!("{}", i);
    i += 1;
}
```

### Mistake 5: Using `for` When You Need `loop`

```rust
// ❌ Awkward: using for just to loop a certain number of times
// (fine, but loop + break is often clearer for retry logic)

// ✅ Use loop when the number of iterations isn't known
let result = loop {
    let attempt = try_something();
    if attempt.is_ok() {
        break attempt;
    }
};
```

### Mistake 6: Expecting while to Return a Value

```rust
// ❌ while loops can't return values
// let x = while condition { break 5; };

// ✅ Use loop instead
let x = loop {
    if some_condition() {
        break 5;
    }
};
```

---

## Exercises

### Exercise 1: Countdown with `while`

Write a countdown from 10 to 1, then print "Liftoff! 🚀". Use a `while` loop.

### Exercise 2: Sum of 1 to N

Write a program using a `for` loop that calculates the sum of all integers from 1 to 100.

Expected result: $\frac{100 \times 101}{2} = 5050$

### Exercise 3: Find First Duplicate

Given the array `[3, 1, 4, 1, 5, 9, 2, 6]`, use a `loop` or `for` to find the first number that appears more than once. Print: "First duplicate: 1".

Hint: Use a second loop or a `Vec` to track seen numbers.

### Exercise 4: Multiplication Table

Print a multiplication table from 1×1 to 5×5 in a formatted grid.

### Exercise 5: FizzBuzz with `for`

Print FizzBuzz for numbers 1 to 50 using a `for` loop:
- Divisible by 3: "Fizz"
- Divisible by 5: "Buzz"
- Divisible by both: "FizzBuzz"
- Otherwise: the number

### Exercise 6: Nested Loop Pattern

Print this pattern using nested `for` loops:

```
*
**
***
****
*****
```

And then this one (inverted):

```
*****
****
***
**
*
```

### Exercise 7: Breaking Out of Nested Loops

Write a program with two nested `for` loops (both 0..10). Use a loop label to break out of BOTH loops when `i * j > 50`.

### Exercise 8: loop with Return Value

Write a `loop` that starts at 1 and keeps doubling. Break when the value exceeds 1000 and return the number of doublings it took. (Expected: 10, since $2^{10} = 1024$)

---

## Summary

### Loop Types

| Loop | Syntax | When to Use |
|------|--------|------------|
| `loop` | `loop { break; }` | Unknown iterations, return values |
| `while` | `while cond { }` | Condition-based |
| `for` | `for x in iter { }` | Collections and ranges |
| `while let` | `while let Some(x) = iter.next() { }` | Pattern-based |

### Control Keywords

| Keyword | Effect |
|---------|--------|
| `break` | Exit the innermost loop |
| `break value` | Exit the loop and return a value (`loop` only) |
| `break 'label` | Exit a specific labeled loop |
| `continue` | Skip to the next iteration |
| `continue 'label` | Skip to the next iteration of a labeled loop |

### Range Syntax

| Syntax | Meaning | Example |
|--------|---------|---------|
| `a..b` | Exclusive range [a, b) | `1..5` → 1, 2, 3, 4 |
| `a..=b` | Inclusive range [a, b] | `1..=5` → 1, 2, 3, 4, 5 |
| `.rev()` | Reverse | `(1..5).rev()` → 4, 3, 2, 1 |
| `.step_by(n)` | Step by n | `(0..10).step_by(2)` → 0, 2, 4, 6, 8 |

### Key Takeaways

1. **`for` is the most common loop** — use it for iterating over collections and ranges
2. **`loop` is for infinite loops** and is the only loop that can return a value via `break`
3. **`while` is for condition-based loops** where you check before each iteration
4. **`break` exits**, **`continue` skips** to the next iteration
5. **Loop labels** (`'outer:`) let you control nested loops
6. **`for` is safer and often faster** than `while` for collections (no bounds checks)
7. **Rust's loops are zero-cost** — they compile to the same machine code as hand-written alternatives
8. **Use `_` for unused iteration variables**: `for _ in 0..5 { }`

---

## What's Next?

We've covered how to store data, organize code, make decisions, and loop. The last piece of fundamentals is learning how to document our code properly!

**Next Tutorial:** [Comments & Code Style →](./08-comments-and-code-style.md)

---

<p align="center">
  <i>Tutorial 7 of 8 — Stage 2: Rust Fundamentals</i>
</p>

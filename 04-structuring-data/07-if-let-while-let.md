# if let & while let 🎯

> **When you only care about one pattern and want to ignore the rest, `if let` and `while let` are cleaner than a full `match`. They're syntactic sugar for matching a single pattern.**

---

## Table of Contents

- [The Problem: Verbose match](#the-problem-verbose-match)
- [if let](#if-let)
  - [With else](#with-else)
  - [With else if let](#with-else-if-let)
- [while let](#while-let)
- [let-else (Rust 1.65+)](#let-else-rust-165)
- [When to Use What](#when-to-use-what)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Problem: Verbose `match`

Sometimes you only care about one variant:

```rust
// Full match — but we only care about Some
let config_max = Some(3u8);
match config_max {
    Some(max) => println!("Max is {}", max),
    _ => (),  // Do nothing for None — feels wasteful
}
```

---

## `if let`

`if let` handles one pattern, ignoring everything else:

```rust
let config_max = Some(3u8);

if let Some(max) = config_max {
    println!("Max is {}", max);
}
// Much cleaner!
```

**Syntax:** `if let PATTERN = EXPRESSION { body }`

The body runs only if the pattern matches.

```rust
// With Option
let name: Option<String> = Some(String::from("Alice"));
if let Some(ref n) = name {
    println!("Hello, {}!", n);
}

// With enum
enum Coin { Penny, Quarter(String) }
let coin = Coin::Quarter(String::from("Alaska"));
if let Coin::Quarter(state) = coin {
    println!("State quarter: {}", state);
}

// With Result
let result: Result<i32, String> = Ok(42);
if let Ok(value) = result {
    println!("Success: {}", value);
}
```

### With `else`

```rust
let color: Option<&str> = None;

if let Some(c) = color {
    println!("Color: {}", c);
} else {
    println!("No color specified, using default");
}
```

### With `else if let`

```rust
enum Animal {
    Dog(String),
    Cat(String),
    Bird,
}

let pet = Animal::Cat(String::from("Whiskers"));

if let Animal::Dog(name) = &pet {
    println!("Dog: {}", name);
} else if let Animal::Cat(name) = &pet {
    println!("Cat: {}", name);
} else {
    println!("Other animal");
}
```

---

## `while let`

Loop as long as a pattern matches:

```rust
// Pop from a stack until empty
let mut stack = vec![1, 2, 3, 4, 5];

while let Some(top) = stack.pop() {
    println!("Popped: {}", top);
}
// Output: 5, 4, 3, 2, 1
println!("Stack is empty!");
```

```rust
// Process a queue
use std::collections::VecDeque;

let mut queue = VecDeque::from([10, 20, 30]);
while let Some(item) = queue.pop_front() {
    println!("Processing: {}", item);
}
```

Common pattern — iterating an iterator manually:

```rust
let mut chars = "hello".chars();
while let Some(c) = chars.next() {
    print!("{} ", c);
}
// h e l l o
```

---

## `let-else` (Rust 1.65+)

`let-else` is the opposite of `if let` — it provides a diverging `else` branch when the pattern doesn't match:

```rust
fn get_count(input: &str) -> u32 {
    let Ok(count) = input.parse::<u32>() else {
        return 0;  // Must diverge: return, break, continue, or panic
    };
    
    count * 2
}

fn main() {
    println!("{}", get_count("5"));    // 10
    println!("{}", get_count("abc"));  // 0
}
```

`let-else` is great for early returns:

```rust
fn process_user(data: Option<&str>) -> String {
    let Some(name) = data else {
        return String::from("Anonymous");
    };
    
    format!("Hello, {}!", name)
}
```

---

## When to Use What

```
┌─────────────────────────────────────────────────────┐
│ Need to handle ALL variants?        → use match     │
│ Only care about ONE variant?        → use if let    │
│ Loop while pattern matches?         → use while let │
│ Bind-or-return-early?              → use let-else  │
│ Simple boolean pattern check?       → use matches!  │
└─────────────────────────────────────────────────────┘
```

| Feature | `match` | `if let` | `while let` | `let-else` |
|---------|---------|----------|-------------|------------|
| Exhaustive | Yes | No | No | No |
| Returns value | Yes | Yes (if/else) | No | N/A |
| Loops | No | No | Yes | No |
| Diverging else | N/A | Optional | N/A | Required |

---

## Exercises

### Exercise 1: Simplify with if let

Rewrite this using `if let`:

```rust
match some_option {
    Some(value) => println!("Got: {}", value),
    None => (),
}
```

<details>
<summary>Solution</summary>

```rust
if let Some(value) = some_option {
    println!("Got: {}", value);
}
```

</details>

### Exercise 2: Drain a Vec

Use `while let` to pop all elements from a Vec, building a reversed string from `vec!['h','e','l','l','o']`.

<details>
<summary>Solution</summary>

```rust
let mut chars = vec!['h', 'e', 'l', 'l', 'o'];
let mut reversed = String::new();

while let Some(c) = chars.pop() {
    reversed.push(c);
}

println!("{}", reversed);  // "olleh"
```

</details>

---

## Summary

```rust
// if let — match one pattern
if let Some(x) = option { use(x); }

// if let with else
if let Ok(v) = result { use(v); } else { handle_err(); }

// while let — loop on a pattern
while let Some(item) = iter.next() { process(item); }

// let-else — bind or diverge
let Some(x) = option else { return; };
```

**Key Takeaways:**
1. `if let` is syntactic sugar for matching a single pattern
2. `while let` loops as long as a pattern matches
3. `let-else` binds or diverges — great for early returns
4. Use `match` when you need exhaustiveness or multiple arms
5. These are convenience features — they don't add new power, just ergonomics

---

**Next:** [Destructuring →](./08-destructuring.md)

<p align="center"><i>Tutorial 7 of 8 — Stage 4: Structuring Data</i></p>

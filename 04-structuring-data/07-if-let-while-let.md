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

### The Ergonomics Story — Why Rust Added These Constructs

Rust's `match` is one of the most powerful features in the language — exhaustive,
expressive, and the backbone of pattern matching. But power comes with ceremony.
When you only care about **one** variant out of many, writing a full `match` with
a `_ => ()` catch-all feels like filling out a government form just to say "yes":

```
┌────────────────────────────────────────────────────────────────┐
│  match value {                                                 │
│      TheOnlyThingICareAbout(x) => do_stuff(x),                │
│      _ => (),   ← boilerplate just to satisfy exhaustiveness  │
│  }                                                             │
└────────────────────────────────────────────────────────────────┘
```

**RFC 160 (2014)** proposed `if let` as *syntactic sugar* — it compiles to
exactly the same code as the `match` + `_ => ()` version, but reads almost
like English:

```rust
// "if value is Some(x), then do_stuff(x)"
if let Some(x) = value {
    do_stuff(x);
}
```

#### Cross-Language Inspiration

| Language       | Single-Pattern Shorthand         | Full Pattern Match        |
|----------------|----------------------------------|---------------------------|
| **Rust**       | `if let Some(x) = val`           | `match val { ... }`       |
| **Swift**      | `if let x = optional`            | `switch val { ... }`      |
| **Haskell**    | *(none — must use `case..of`)*   | `case val of { ... }`     |
| **Python 3.10**| *(none — full match required)*   | `match val: case ...`     |
| **Kotlin**     | `val x = expr as? Type`          | `when(val) { ... }`       |

Swift had `if let` from its **1.0 release (June 2014)**, and Rust adopted the
same concept shortly after. The Rust community debated whether adding another
way to do the same thing would fragment code style, but the verdict was clear:
**readability wins**.

Haskell, despite being a pattern-matching pioneer, has no shorthand equivalent —
you always write `case expr of`. Python 3.10 added structural pattern matching
in 2021, but likewise only provides the full `match` statement with no single-
pattern shortcut.

The design principle at work:

```
╔══════════════════════════════════════════════════════════════╗
║  "Make the common case easy and the complex case possible." ║
╚══════════════════════════════════════════════════════════════╝
```

`match` keeps the complex case possible (and safe, via exhaustiveness).
`if let` makes the common case — caring about just one variant — *easy*.

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

### `let-else` — The Missing Piece (Rust 1.65)

`let-else` was stabilized in **Rust 1.65 (November 2022)** and was one of the
most requested features in the language. Before it arrived, the "guard clause"
pattern — *validate input, bail early on failure* — required awkward workarounds:

```rust
// ── BEFORE let-else ──────────────────────────────────────────

// Option 1: match with early return (verbose)
fn option1(input: Option<u32>) -> u32 {
    let value = match input {
        Some(v) => v,
        None => return 0,
    };
    value * 2
}

// Option 2: if let with happy path INSIDE the block
//           (inverted logic → "rightward drift")
fn option2(input: Option<u32>) -> u32 {
    if let Some(value) = input {
        value * 2       // ← real logic is indented
    } else {
        0
    }
}

// Option 3: unwrap / expect (panics on None — bad for production)
fn option3(input: Option<u32>) -> u32 {
    let value = input.expect("must be Some"); // 💥 panics
    value * 2
}

// ── AFTER let-else ───────────────────────────────────────────
fn clean(input: Option<u32>) -> u32 {
    let Some(value) = input else { return 0; };
    value * 2   // ← no nesting, value is always valid here
}
```

#### The Divergence Requirement

The `else` block **must diverge** — it cannot fall through. The compiler
accepts only these:

```
┌──────────────────────────────────────────────────────┐
│  return expr;     ← exit the function                │
│  break;           ← exit a loop                      │
│  continue;        ← skip to next iteration           │
│  panic!("...");   ← abort the program                │
│  loop { }         ← loop forever (type = !)          │
│  std::process::exit(1);                              │
└──────────────────────────────────────────────────────┘
```

Because the else block *always* diverges, the compiler **guarantees** that
after the `let-else` line, the binding is valid and fully initialized — no
`Option<T>` wrapper, no unwrapping, just the inner value.

#### Cross-Language Parallels

| Language   | Guard Pattern                                          |
|------------|--------------------------------------------------------|
| **Rust**   | `let Some(x) = expr else { return; };`                 |
| **Swift**  | `guard let x = expr else { return }`                   |
| **Go**     | `val, ok := expr; if !ok { return }`                   |
| **Kotlin** | `val x = expr ?: return`                               |

Swift's `guard let` was the direct inspiration. Go achieves the same effect
with its comma-ok idiom, but without destructuring. Kotlin uses the Elvis
operator `?:` for a concise version.

#### Real-World Impact

`let-else` **significantly reduces nesting** in functions that validate
multiple inputs. Compare the before/after for a function with three checks:

```
  BEFORE (if let nesting)          AFTER (let-else flat)
  ┌───────────────────────┐        ┌───────────────────────┐
  │ if let Some(a) = x {  │        │ let Some(a) = x else  │
  │   if let Ok(b) = y {  │        │   { return Err(..); };│
  │     if let Some(c) = z│        │ let Ok(b) = y else    │
  │     {                  │        │   { return Err(..); };│
  │       // finally here  │        │ let Some(c) = z else  │
  │     }                  │        │   { return Err(..); };│
  │   }                    │        │                       │
  │ }                      │        │ // all bindings valid  │
  │ // 3 levels deep       │        │ // zero nesting        │
  └───────────────────────┘        └───────────────────────┘
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

### Pattern Matching Ergonomics Summary — Choosing the Right Tool

Rust offers a family of pattern-matching constructs, each tuned for a specific
scenario. Picking the right one makes your code shorter, clearer, and more
idiomatic.

#### Decision Flowchart

```
                  ┌──────────────────────┐
                  │ How many patterns do  │
                  │ you need to handle?   │
                  └─────────┬────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
           ONE          MULTIPLE      BOOLEAN
              │             │         CHECK ONLY
              │             │             │
     ┌────────┴───────┐     │             ▼
     ▼                ▼     ▼       matches!() macro
  Need to bind     In a    match
  or bail early?   loop?
     │                │
     ▼                ▼
  let-else        while let
  (guard clause)
     │
     ▼
  if let
  (optional action)
```

#### Quick-Reference Table

| Construct    | Use Case                                        | Exhaustive? | Expression? | Since    |
|-------------|--------------------------------------------------|-------------|-------------|----------|
| `match`     | Handle multiple variants; need exhaustiveness     | **Yes**     | Yes         | 1.0      |
| `if let`    | Care about one variant; optionally do something   | No          | Yes (w/ else)| 1.0     |
| `while let` | Consume items from iterator/channel returning Option | No       | No          | 1.0      |
| `let-else`  | Guard clause — bind or bail early                 | No          | N/A         | **1.65** |
| `matches!()` | Boolean check — use in `.filter()`, `assert!`, etc. | No       | Yes (bool)  | 1.42     |

#### The `matches!()` Macro

Often overlooked, `matches!()` returns a `bool` and is perfect when you don't
need the inner value — just a yes/no answer:

```rust
let val = Some(3);
assert!(matches!(val, Some(1..=5)));  // true — val is Some of 1,2,3,4, or 5

// Useful in iterator chains:
let items = vec![Some(1), None, Some(4), Some(10)];
let small: Vec<_> = items.iter()
    .filter(|x| matches!(x, Some(1..=5)))
    .collect();
// [Some(1), Some(4)]
```

#### The Evolution Timeline

```
  Rust 1.0 (2015)          Rust 1.42 (2020)       Rust 1.65 (2022)     Nightly / Future
  ───────────────          ────────────────        ────────────────     ──────────────────
  match                    matches!() macro        let-else             if let chains
  if let                                                                (if let && let)
  while let
```

**Coming soon — `if let` chains** (currently on nightly):

```rust
// Combine multiple let patterns with boolean conditions
// (unstable — requires #![feature(let_chains)])
if let Some(x) = opt_a && let Ok(y) = res_b && x > 0 {
    println!("x = {x}, y = {y}");
}
```

This will further reduce nesting for multi-condition checks. Until it
stabilizes, use `let-else` followed by `if` for the same effect.

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

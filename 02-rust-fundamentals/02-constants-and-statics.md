# Constants & Statics 🔒

> **Constants and statics are values that never change — but they serve different purposes and live in different places.**

---

## Table of Contents

- [Why Do We Need Constants?](#why-do-we-need-constants)
- [Creating Constants with const](#creating-constants-with-const)
- [const vs let — What's the Difference?](#const-vs-let--whats-the-difference)
- [Constant Expressions — What Can Be a const?](#constant-expressions--what-can-be-a-const)
- [Naming Convention — SCREAMING_SNAKE_CASE](#naming-convention--screaming_snake_case)
- [Where Can Constants Be Declared?](#where-can-constants-be-declared)
- [Constants in Memory](#constants-in-memory)
- [Static Variables with static](#static-variables-with-static)
- [const vs static — When to Use Which](#const-vs-static--when-to-use-which)
- [Mutable Statics — Danger Zone](#mutable-statics--danger-zone)
- [Real-World Examples](#real-world-examples)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Why Do We Need Constants?

Imagine you're writing a game. You have values like:

- Maximum player health: **100**
- Screen width: **1920**
- Gravity: **9.81**

These values:
1. **Never change** during the program
2. **Have meaning** — `100` alone is meaningless, but `MAX_HEALTH` is clear
3. **Might be used in many places** — changing the value should only need one edit

Constants solve all three problems by giving permanent, descriptive names to fixed values.

### The History of Constants in Programming

The concept of named constants evolved through programming history:

- **Assembly language (1950s):** Programmers used **EQU** (equate) directives to give names to numbers: `MAX_SIZE EQU 100`. This was purely for readability — the assembler substituted the number during assembly.

- **C (1972):** Originally used `#define MAX_SIZE 100` — a preprocessor text substitution, not a true constant. The preprocessor literally replaced `MAX_SIZE` with `100` before the compiler even saw it. This caused subtle bugs because `#define` has no type checking and no scope. C99 later added `const`, but C's `const` is weaker than Rust's — a C `const` variable still has a runtime address and can be modified through pointers (undefined behavior, but possible).

- **Java (1995):** Uses `final` for constants: `final int MAX_SIZE = 100;`. Java's `final` prevents reassignment but doesn't guarantee compile-time evaluation.

- **C++ (2011):** Added `constexpr` — similar to Rust's `const`, it guarantees compile-time evaluation. Rust's `const` and `const fn` were directly influenced by C++'s `constexpr`.

- **JavaScript (2015):** Added `const` in ES6, but it only prevents reassignment — the value itself can still be mutated if it's an object: `const arr = [1,2,3]; arr.push(4);` works! This is NOT what most people expect from "constant."

- **Rust:** Takes the strongest position: `const` means truly constant — evaluated at compile time, inlined everywhere, no runtime existence. This is as close to mathematical constants as you can get in a programming language.

| Language | Keyword | Compile-time? | Type-safe? | True constant? |
|----------|---------|---------------|------------|----------------|
| C | `#define` | Preprocessor | ❌ No | ❌ Text substitution |
| C | `const` | ❌ Runtime | ✅ Yes | ⚠️ Weak (can be cast away) |
| C++ | `constexpr` | ✅ Yes | ✅ Yes | ✅ Yes |
| Java | `final` | ❌ Runtime | ✅ Yes | ⚠️ Only prevents reassignment |
| JavaScript | `const` | ❌ Runtime | ❌ Dynamic | ⚠️ Only prevents reassignment |
| Python | UPPERCASE | ❌ Convention | ❌ Dynamic | ❌ Just a naming convention |
| Rust | `const` | ✅ Yes | ✅ Yes | ✅ Yes |

---

## Creating Constants with `const`

```rust
const MAX_HEALTH: i32 = 100;
const SCREEN_WIDTH: u32 = 1920;
const GRAVITY: f64 = 9.81;
const GAME_TITLE: &str = "Rust Quest";
const IS_DEBUG: bool = false;

fn main() {
    println!("Welcome to {}!", GAME_TITLE);
    println!("Max health: {}", MAX_HEALTH);
    println!("Gravity: {} m/s²", GRAVITY);
}
```

Output:
```
Welcome to Rust Quest!
Max health: 100
Gravity: 9.81 m/s²
```

### The Syntax

```
const NAME: Type = value;
│     │     │       │
│     │     │       └── The value (must be known at compile time)
│     │     └────────── The type (REQUIRED — no type inference for const)
│     └──────────────── The name (SCREAMING_SNAKE_CASE)
└────────────────────── The keyword
```

### Important Rule: Type Annotation is REQUIRED

Unlike `let`, you **must** always write the type for `const`:

```rust
const X: i32 = 5;     // ✅ Type annotation required
// const X = 5;        // ❌ ERROR: missing type for `const` item
```

Why? Because constants have no runtime existence — the compiler substitutes the value directly everywhere the constant is used. Having an explicit type makes the code clearer and prevents subtle type errors across large codebases.

---

## const vs let — What's the Difference?

| Feature | `let` | `const` |
|---------|-------|---------|
| Keyword | `let x = 5;` | `const X: i32 = 5;` |
| Type annotation | Optional | **Required** |
| Naming convention | `snake_case` | `SCREAMING_SNAKE_CASE` |
| Can use `mut`? | ✅ Yes | ❌ Never |
| When is value evaluated? | At runtime | At **compile time** |
| Where can it be declared? | Inside functions only | Anywhere (inside or outside functions) |
| Shadowing? | ✅ Yes | ❌ No |
| Scope | Block scope `{}` | Depends on where declared |
| Memory | Lives on the stack | **Inlined** (no fixed memory address) |

### The Compile-Time Difference

This is the crucial distinction. `const` values must be computable at **compile time** — before the program runs:

```rust
const SECONDS_IN_HOUR: u32 = 60 * 60;           // ✅ Compiler can calculate this
const SECONDS_IN_DAY: u32 = SECONDS_IN_HOUR * 24; // ✅ Uses another const

fn main() {
    let user_input = get_user_input();  // value unknown until runtime
    // const X: i32 = user_input;       // ❌ ERROR: const can't use runtime values
    let x = user_input;                 // ✅ let is fine with runtime values
}
```

Think of it this way:
```
const → "I know this value while writing the code"
let   → "I'll find out this value when the program runs"
```

---

## Constant Expressions — What Can Be a `const`?

The value assigned to a `const` must be a **constant expression** — something the compiler can evaluate at compile time.

### ✅ Valid Constant Expressions

```rust
// Literals
const A: i32 = 42;
const B: f64 = 3.14;
const C: bool = true;
const D: char = 'Z';
const E: &str = "hello";

// Arithmetic operations
const SUM: i32 = 10 + 20;
const PRODUCT: i32 = 6 * 7;
const POWER: u32 = 2_u32.pow(10);  // 1024

// Combining other constants
const BASE: i32 = 100;
const DOUBLED: i32 = BASE * 2;     // 200

// Boolean expressions
const IS_BIG: bool = BASE > 50;    // true

// Tuple of constants
const POINT: (i32, i32) = (10, 20);

// Array of constants
const PRIMES: [i32; 5] = [2, 3, 5, 7, 11];

// String byte length
const GREETING: &str = "Hello";
const GREETING_LEN: usize = GREETING.len();  // 5
```

### ❌ Invalid Constant Expressions

```rust
// Function calls that aren't const fn (more on this later)
// const X: String = String::from("hello");  // ❌ String::from is not const

// Using variables
// let y = 5;
// const Z: i32 = y;  // ❌ Can't use runtime variables

// User input
// const NAME: &str = read_line();  // ❌ Can't call non-const functions
```

### What's a `const fn`?

Some functions are marked `const fn`, meaning they can be evaluated at compile time:

```rust
const fn add(a: i32, b: i32) -> i32 {
    a + b
}

const RESULT: i32 = add(3, 4);  // ✅ 7 — evaluated at compile time

fn main() {
    println!("Result: {}", RESULT);  // Result: 7
    
    // const fn can also be called at runtime
    let runtime_result = add(5, 6);  // ✅ Also works at runtime
    println!("Runtime: {}", runtime_result);
}
```

> **Note:** `const fn` has restrictions — it can't use heap allocation, trait objects, or many other runtime features. The full details are beyond this stage, but know that basic math and logic work fine.

---

## Naming Convention — SCREAMING_SNAKE_CASE

Constants in Rust use **SCREAMING_SNAKE_CASE** — all uppercase with underscores:

```rust
// ✅ GOOD — SCREAMING_SNAKE_CASE
const MAX_PLAYERS: u32 = 4;
const PI: f64 = 3.14159265358979;
const DEFAULT_TIMEOUT_MS: u64 = 5000;
const HTTP_STATUS_OK: u16 = 200;

// ❌ BAD — the compiler will warn you
const max_players: u32 = 4;     // ⚠️ should be MAX_PLAYERS
const MaxPlayers: u32 = 4;      // ⚠️ should be MAX_PLAYERS
const maxPlayers: u32 = 4;      // ⚠️ should be MAX_PLAYERS
```

This convention exists across many programming languages (C, Java, Python, etc.) and makes constants visually distinct from regular variables in your code.

---

## Where Can Constants Be Declared?

### 1. Outside Functions (Module Level)

```rust
const MAX_SIZE: usize = 1024;  // Available throughout the file

fn main() {
    println!("Max size: {}", MAX_SIZE);  // ✅
}

fn other_function() {
    println!("Also here: {}", MAX_SIZE);  // ✅
}
```

### 2. Inside Functions

```rust
fn main() {
    const LOCAL_LIMIT: i32 = 50;
    println!("Limit: {}", LOCAL_LIMIT);  // ✅
}

fn other_function() {
    // println!("{}", LOCAL_LIMIT);  // ❌ Not accessible here
}
```

### 3. In Modules (we'll cover modules later)

```rust
mod config {
    pub const API_URL: &str = "https://api.example.com";
    pub const MAX_RETRIES: u32 = 3;
}

fn main() {
    println!("URL: {}", config::API_URL);
}
```

> **Real-World Practice:** Most constants are declared at the top of a file (outside functions) or in a dedicated `constants.rs` file. This makes them easy to find and change.

---

## Constants in Memory

Here's something interesting — constants don't have a fixed memory address. The compiler **inlines** them:

```rust
const MAX: i32 = 100;

fn main() {
    let a = MAX;       // Compiler replaces with: let a = 100;
    let b = MAX + 50;  // Compiler replaces with: let b = 100 + 50;
    
    if MAX > 0 {       // Compiler replaces with: if 100 > 0 { (always true)
        println!("Positive!");
    }
}
```

What the compiler actually sees after inlining:

```rust
fn main() {
    let a = 100;
    let b = 150;        // Pre-computed!
    
    // The if is eliminated entirely (compiler knows 100 > 0 is always true)
    println!("Positive!");
}
```

This means:
- **No memory is allocated** for the constant itself
- Every use of the constant is replaced with the literal value
- The compiler can optimize aggressively (constant folding, dead code elimination)
- You **cannot take the address** of a constant in a meaningful way

### Deep Dive: Constant Folding and Propagation

What the compiler does with constants is part of a family of optimizations called **constant folding** and **constant propagation**:

**Constant folding** — evaluating constant expressions at compile time:
```rust
const A: i32 = 10;
const B: i32 = A * 2 + 5;  // Compiler computes: 25
// In the binary, B is just the number 25 — no multiplication happens at runtime
```

**Constant propagation** — replacing variable references with their known values:
```rust
const MAX: i32 = 100;
let x = MAX + 50;  
// Compiler sees: let x = 100 + 50;
// Then constant folds: let x = 150;
// The final binary just loads 150 directly
```

**Dead code elimination** — removing code that can never execute:
```rust
const DEBUG: bool = false;

if DEBUG {
    println!("Debug info...");  // This ENTIRE block is removed from the binary
}
```

These optimizations mean that using `const` has **zero runtime cost** — in fact, it can make your code *faster* than using variables, because the compiler has more information to work with.

> **Fun Fact:** Modern compilers are remarkably good at these optimizations. LLVM (the compiler backend Rust uses) can evaluate complex expressions involving hundreds of operations at compile time. Rust's `const fn` feature lets you write arbitrarily complex computations that execute entirely during compilation.

---

## Static Variables with `static`

Now let's look at `static` — a different way to create a fixed value:

```rust
static GREETING: &str = "Hello, World!";
static MAX_THREADS: u32 = 8;
static VERSION: &str = "1.0.0";

fn main() {
    println!("{}", GREETING);
    println!("Max threads: {}", MAX_THREADS);
    println!("Version: {}", VERSION);
}
```

### The Syntax

```
static NAME: Type = value;
│      │     │       │
│      │     │       └── The value
│      │     └────────── The type (REQUIRED)
│      └──────────────── The name (SCREAMING_SNAKE_CASE)
└────────────────────── The keyword
```

### How `static` Differs from `const`

A `static` variable has a **fixed memory address** that lasts for the entire program:

```
Program Memory Layout:

┌──────────────┐
│   Code       │  ← Your compiled functions
├──────────────┤
│   Static     │  ← static variables live HERE
│   Data       │     (fixed address, exists for entire program)
├──────────────┤
│   Stack      │  ← let variables live here (per-function)
├──────────────┤
│   Heap       │  ← Box, Vec, String allocations (dynamic)
└──────────────┘
```

When you use a `static`:
```rust
static X: i32 = 42;

fn main() {
    // X exists at a specific memory address, say 0x7F00
    let reference = &X;  // ✅ You CAN take the address of a static
    println!("Value: {}, Address: {:p}", X, reference);
}
```

### 'static Lifetime

Static variables have the `'static` lifetime, meaning they exist from program start to program end:

```
Program timeline:
─────────────────────────────────────────────
│ START │          RUNNING               │ END │
─────────────────────────────────────────────
│       │←── static variables exist here ──→│
│       │                                    │
│  ┌──┐ │                              ┌──┐ │
│  │fn│ │  ← stack variables exist     │fn│ │
│  └──┘ │    only during function call  └──┘ │
```

We'll cover lifetimes in depth in Stage 5. For now, just know that `static` means "lives forever."

### Deep Dive: Where Static Data Actually Lives

When your operating system loads a program into memory, it creates several **segments** (regions of memory with different purposes):

```
Program Memory Map (simplified):
┌──────────────────────┐  High addresses
│                      │
│       Stack          │  ← Local variables (grows downward ↓)
│         ↓            │
│                      │
│       (gap)          │
│                      │
│         ↑            │
│       Heap           │  ← Dynamic allocations (grows upward ↑)
│                      │
├──────────────────────┤
│   BSS Segment        │  ← Uninitialized static data (zeroed)
├──────────────────────┤
│   Data Segment       │  ← Initialized static data (your `static` variables!)
├──────────────────────┤
│   Read-Only Data     │  ← String literals, const data embedded in binary
├──────────────────────┤
│   Text (Code)        │  ← Your compiled machine code
└──────────────────────┘  Low addresses
```

When you write `static GREETING: &str = "Hello, World!"`:
- The string bytes `"Hello, World!"` go into the **Read-Only Data** segment
- The `static` variable (a pointer + length) goes into the **Data Segment**
- Both are embedded directly in your compiled binary (the `.exe` or ELF file)
- When the OS loads your program, these segments are memory-mapped from the file

This is why static data has a fixed address — the address is determined when the program is compiled (or more precisely, when it's linked) and doesn't change during execution. It's also why `static` variables exist for the entire program lifetime — they're part of the binary itself.

> **Comparison with Other Languages:**
> - **C/C++:** Global and static variables work the same way — stored in the data segment. Rust adopted this directly from C.
> - **Java:** Has no true static data segment — everything lives on the heap, managed by the garbage collector. "Static" fields in Java are just class-level fields, not memory segment residents.
> - **Python:** Everything is a heap-allocated object. Python has no concept of compile-time static data.

---

## const vs static — When to Use Which

| Feature | `const` | `static` |
|---------|---------|----------|
| Has fixed memory address? | ❌ No (inlined) | ✅ Yes |
| Can take a reference `&`? | Technically yes (creates temp) | ✅ Yes (real address) |
| Lives for entire program? | N/A (no single location) | ✅ Yes |
| Performance | Values inlined (potentially faster) | Single location (saves space for large data) |
| Can be mutable? | ❌ Never | ⚠️ Yes, but `unsafe` required |
| Use when | Small values, math constants | Large data, FFI, unique address needed |

### Rule of Thumb

```
Use const for:
  → Mathematical constants (PI, E, MAX_RETRIES)
  → Configuration values (BUFFER_SIZE, TIMEOUT_MS)
  → Anything where you just need a named value
  → 99% of the time, const is what you want

Use static for:
  → When you need a fixed memory address
  → Large data that shouldn't be duplicated at every use site
  → When working with C libraries (FFI)
  → Global state (rare, usually a code smell)
```

### Example: When `static` Saves Memory

```rust
// const: the 1000-element array is COPIED everywhere it's used
const BIG_TABLE: [i32; 1000] = [0; 1000];
// Every function that uses BIG_TABLE gets its own copy in the binary!

// static: the array exists ONCE in memory, shared by all references
static BIG_TABLE_STATIC: [i32; 1000] = [0; 1000];
// Every function that uses it references the SAME memory
```

For small values (integers, bools, etc.), `const` is better because inlining is faster than reading from a fixed address. For large data, `static` can save binary size.

---

## Mutable Statics — Danger Zone

You can technically create a mutable static variable, but it requires `unsafe`:

```rust
static mut COUNTER: i32 = 0;

fn main() {
    unsafe {
        COUNTER += 1;
        COUNTER += 1;
        COUNTER += 1;
        println!("Counter: {}", COUNTER);  // Counter: 3
    }
}
```

### Why is `unsafe` Required?

Mutable global state is **dangerous** because:

1. **Data races** — If two threads modify `COUNTER` simultaneously, the result is undefined
2. **No synchronization** — Rust can't guarantee safe access across threads
3. **Hard to reason about** — Any code anywhere could modify it

```
Thread 1: reads COUNTER (value: 5)
Thread 2: reads COUNTER (value: 5)
Thread 1: writes COUNTER = 5 + 1 = 6
Thread 2: writes COUNTER = 5 + 1 = 6  ← Should be 7!
```

> **⚠️ WARNING:** Mutable statics are almost NEVER the right solution. In real Rust code, you'd use:
> - `Mutex<T>` for thread-safe mutable state
> - `AtomicI32` for atomic operations
> - `lazy_static!` or `once_cell` for lazy initialization
>
> We'll cover these safer alternatives in Stages 12-13 (Concurrency). For now, just know that `static mut` exists but should be avoided.

---

## Real-World Examples

### Configuration Constants

```rust
// In a web server
const MAX_CONNECTIONS: usize = 1000;
const DEFAULT_PORT: u16 = 8080;
const REQUEST_TIMEOUT_SECS: u64 = 30;
const MAX_BODY_SIZE: usize = 10 * 1024 * 1024;  // 10 MB
const API_VERSION: &str = "v2";

// In a game
const FPS: u32 = 60;
const TILE_SIZE: u32 = 32;
const MAX_ENTITIES: usize = 10_000;
const GRAVITY: f64 = -9.81;
const JUMP_FORCE: f64 = 15.0;
```

### Mathematical Constants

```rust
const PI: f64 = std::f64::consts::PI;  // Using the standard library's PI
const TAU: f64 = PI * 2.0;
const E: f64 = std::f64::consts::E;
const GOLDEN_RATIO: f64 = 1.618033988749895;
const DEG_TO_RAD: f64 = PI / 180.0;
```

> **Tip:** Rust's standard library provides many mathematical constants in `std::f64::consts`. Use those instead of defining your own!

### Using Constants for Readable Code

```rust
const SECONDS_PER_MINUTE: u64 = 60;
const MINUTES_PER_HOUR: u64 = 60;
const HOURS_PER_DAY: u64 = 24;
const DAYS_PER_YEAR: u64 = 365;

const SECONDS_PER_HOUR: u64 = SECONDS_PER_MINUTE * MINUTES_PER_HOUR;    // 3,600
const SECONDS_PER_DAY: u64 = SECONDS_PER_HOUR * HOURS_PER_DAY;          // 86,400
const SECONDS_PER_YEAR: u64 = SECONDS_PER_DAY * DAYS_PER_YEAR;          // 31,536,000

fn main() {
    let uptime_seconds: u64 = 172_800;
    let days = uptime_seconds / SECONDS_PER_DAY;
    let remaining = uptime_seconds % SECONDS_PER_DAY;
    let hours = remaining / SECONDS_PER_HOUR;
    
    println!("Uptime: {} days, {} hours", days, hours);
    // Uptime: 2 days, 0 hours
}
```

Without constants, this would be `172800 / 86400` — completely unreadable.

---

## Common Mistakes

### Mistake 1: Forgetting the Type Annotation

```rust
// const X = 5;  // ❌ ERROR: missing type for `const` item
const X: i32 = 5;  // ✅
```

### Mistake 2: Trying to Make a `const` from a Runtime Value

```rust
fn main() {
    let input = read_input();
    // const RESULT: i32 = input;  // ❌ ERROR: attempt to use a non-constant value
    let result = input;            // ✅ Use let for runtime values
}
```

### Mistake 3: Using lowercase for Constants

```rust
// const max_size: usize = 100;  // ⚠️ Warning: should be SCREAMING_SNAKE_CASE
const MAX_SIZE: usize = 100;     // ✅
```

### Mistake 4: Confusing const, static, and let

```rust
// Quick reference:
let x = 5;                // Runtime variable, immutable, block scoped
let mut x = 5;            // Runtime variable, mutable, block scoped
const X: i32 = 5;         // Compile-time constant, inlined, no address
static X: i32 = 5;        // Fixed address, lives entire program, immutable
static mut X: i32 = 5;    // Mutable global — requires unsafe, avoid this
```

---

## Exercises

### Exercise 1: Configuration Constants

Define constants for a simple web application:
- Maximum username length (30 characters)
- Minimum password length (8 characters)
- Maximum login attempts (5)
- Session timeout in seconds (3600 = 1 hour)
- Application name ("MyApp")

Print all of them in the format:
```
=== MyApp Configuration ===
Max username length: 30
Min password length: 8
Max login attempts: 5
Session timeout: 3600 seconds
```

### Exercise 2: Temperature Conversion

Define a constant for the Fahrenheit-to-Celsius conversion and write a program that converts 100°F to Celsius:

Formula: `C = (F - 32) × 5/9`

```
100°F = 37.78°C
```

### Exercise 3: const fn

Write a `const fn` called `circle_area_approx` that takes a radius (as `i32`) and returns an approximate area (as `i32`), using a constant `PI_APPROX: i32 = 3`.

Use this function to define a `const AREA: i32` with a radius of 10.

### Exercise 4: Identify the Error

Which of these are valid? Explain why the invalid ones fail:

```rust
const A: i32 = 5;                    // 1
const B = 10;                        // 2
const C: i32 = A + 1;                // 3
static D: i32 = 5;                   // 4
static mut E: i32 = 5;               // 5
const F: i32 = some_function();      // 6 (assume some_function is NOT const fn)
```

---

## Summary

| Feature | `const` | `static` | `let` |
|---------|---------|----------|-------|
| Keyword | `const` | `static` | `let` |
| Type required? | ✅ Yes | ✅ Yes | ❌ Optional |
| Value known at | Compile time | Compile time | Runtime OK |
| Memory | Inlined (no address) | Fixed address | Stack |
| Lifetime | N/A | Entire program | Until scope ends |
| Mutable? | ❌ Never | ⚠️ With unsafe | ✅ With mut |
| Naming | `SCREAMING_SNAKE_CASE` | `SCREAMING_SNAKE_CASE` | `snake_case` |
| Use for | Named constants | Global data with address | Everything else |

### Key Takeaways

1. **Use `const` for 99% of named constants** — it's the simplest and most optimized
2. **Type annotation is required** for both `const` and `static`
3. **SCREAMING_SNAKE_CASE** is the naming convention for both
4. **`const` values are inlined** — the compiler pastes the value everywhere
5. **`static` values have a fixed address** — use for large data or FFI
6. **Avoid `static mut`** — it requires `unsafe` and is almost never the right solution
7. **`const fn`** allows functions to be evaluated at compile time

---

## What's Next?

Now that we know how to store values, let's learn about ALL the types of values Rust can store — starting with the simple ones (scalars).

**Next Tutorial:** [Scalar Data Types →](./03-scalar-data-types.md)

---

<p align="center">
  <i>Tutorial 2 of 8 — Stage 2: Rust Fundamentals</i>
</p>

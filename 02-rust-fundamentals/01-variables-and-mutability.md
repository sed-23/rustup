# Variables & Mutability 📦

> **Variables are containers for data. In Rust, they're immutable by default — and that's a feature, not a bug.**

---

## Table of Contents

- [What is a Variable?](#what-is-a-variable)
- [Creating Variables with let](#creating-variables-with-let)
- [What Happens in Memory?](#what-happens-in-memory)
- [Type Inference — Rust Guesses the Type](#type-inference--rust-guesses-the-type)
- [Type Annotations — Telling Rust the Type](#type-annotations--telling-rust-the-type)
- [Immutability by Default](#immutability-by-default)
- [Making Variables Mutable with mut](#making-variables-mutable-with-mut)
- [Why Immutable by Default?](#why-immutable-by-default)
- [Shadowing — Reusing a Variable Name](#shadowing--reusing-a-variable-name)
- [Shadowing vs mut — What's the Difference?](#shadowing-vs-mut--whats-the-difference)
- [Variable Scope — Where Variables Live](#variable-scope--where-variables-live)
- [Unused Variables](#unused-variables)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What is a Variable?

A variable is a **named storage location** in your computer's memory. Think of it as a labeled box:

```
┌─────────┐
│   42    │  ← The value stored inside
└─────────┘
    x        ← The label (variable name)
```

When you create a variable, three things happen:

1. **Memory is reserved** — space is set aside to hold the value
2. **The value is stored** — the data is written into that memory
3. **The name is bound** — the name `x` is associated with that memory location

---

## Creating Variables with `let`

In Rust, you create variables using the `let` keyword:

```rust
fn main() {
    let x = 5;
    println!("The value of x is: {}", x);
}
```

Output:
```
The value of x is: 5
```

### Breaking It Down

```
let x = 5;
│   │   │ │
│   │   │ └── Semicolon: this statement is complete
│   │   └──── The value we're storing (an integer literal)
│   │  
│   └──────── The variable name (also called a "binding")
└──────────── The keyword that creates a variable
```

### What Does "Binding" Mean?

In Rust, we say `let x = 5` **binds** the name `x` to the value `5`. This is more than pedantic — in Rust, `let` doesn't just declare a variable, it creates a *binding* between a name and a value. The distinction becomes important when we learn about ownership in Stage 3.

```
"x is bound to 5"    ← Rust terminology
"x contains 5"       ← Also fine for now
"x equals 5"         ← Casual, works for beginners
```

### Creating Multiple Variables

```rust
fn main() {
    let name = "Alice";
    let age = 30;
    let height = 5.6;
    let is_student = false;
    
    println!("{} is {} years old", name, age);
    println!("Height: {} feet, Student: {}", height, is_student);
}
```

Output:
```
Alice is 30 years old
Height: 5.6 feet, Student: false
```

### Variable Naming Rules

Rust variable names must follow these rules:

| Rule | Example (Valid) | Example (Invalid) |
|------|----------------|-------------------|
| Must start with a letter or underscore | `age`, `_temp` | `1name`, `@data` |
| Can contain letters, digits, underscores | `user_name`, `count2` | `user-name`, `my.var` |
| Must use **snake_case** (convention) | `first_name`, `total_count` | `firstName`, `TotalCount` |
| Cannot use reserved keywords | `my_let`, `my_fn` | `let`, `fn`, `struct` |
| Case-sensitive | `name` ≠ `Name` ≠ `NAME` | — |

```rust
// GOOD — snake_case is the Rust convention
let first_name = "Alice";
let total_count = 42;
let is_valid = true;
let _unused = 0;    // Underscore prefix = "I know this is unused"

// BAD — the compiler will warn you (but it still works)
let firstName = "Alice";    // ⚠️ Warning: should use snake_case
let TotalCount = 42;        // ⚠️ Warning: should use snake_case
```

> **Real-World Practice:** Rust enforces `snake_case` for variable and function names via a compiler warning. The entire Rust ecosystem follows this convention. If you're coming from Java or JavaScript (camelCase), this will take a few days to get used to.

---

## What Happens in Memory?

When you write `let x = 5;`, here's what happens on the **stack** (we'll explain stack vs heap in Stage 3):

```
Stack Memory (simplified):
┌───────────────┐
│ x = 5         │  ← 4 bytes reserved, value 5 stored
├───────────────┤
│ (next var)    │
├───────────────┤
│ ...           │
└───────────────┘
```

### How Much Memory?

Different types use different amounts of memory:

```rust
let a: i32 = 5;       // 4 bytes (32 bits)
let b: i64 = 5;       // 8 bytes (64 bits)
let c: bool = true;   // 1 byte
let d: char = 'A';    // 4 bytes (Unicode scalar value)
let e: f64 = 3.14;    // 8 bytes (64-bit float)
```

We'll cover all types in detail in Tutorials 3 and 4 of this stage.

---

## Type Inference — Rust Guesses the Type

Rust is a **statically typed** language — every variable has a type known at compile time. But you don't always have to write the type explicitly! Rust has **type inference**:

```rust
fn main() {
    let x = 5;          // Rust infers: i32 (32-bit integer)
    let y = 3.14;       // Rust infers: f64 (64-bit float)
    let z = true;       // Rust infers: bool
    let name = "Alice"; // Rust infers: &str (string slice)
    let c = 'A';        // Rust infers: char
}
```

### How Does Rust Know?

The compiler looks at:
1. **The value you assigned** — `5` is an integer, `3.14` is a float
2. **How you USE the variable later** — if you pass it to a function expecting `u8`, the compiler knows
3. **The default** — when ambiguous, integers default to `i32`, floats default to `f64`

### When Type Inference Fails

Sometimes Rust can't figure out the type and needs help:

```rust
fn main() {
    // ❌ This won't compile — Rust doesn't know which type of collection
    let numbers = Vec::new();  // Vec of what? i32? String? 
    
    // ✅ Fix: tell Rust the type
    let numbers: Vec<i32> = Vec::new();  // A vector of 32-bit integers
    
    // ✅ Or let Rust figure it out from usage:
    let mut numbers = Vec::new();
    numbers.push(5);  // Now Rust knows: Vec<i32> (because 5 is i32)
}
```

### What Does "Statically Typed" Mean?

```
Statically typed (Rust, C, Java, TypeScript):
  → Every variable's type is known at COMPILE time
  → Type mismatches are caught BEFORE the program runs
  → Once a variable has a type, it CANNOT change

Dynamically typed (Python, JavaScript, Ruby):
  → Types are checked at RUNTIME
  → Type errors appear when the code runs
  → A variable can hold any type at any time
```

```rust
// In Rust, a variable's type is FIXED
let x = 5;       // x is i32
// x = "hello";  // ❌ ERROR: can't assign a string to an i32 variable

// In Python, this would work fine:
// x = 5
// x = "hello"   # No error in Python!
```

---

## Type Annotations — Telling Rust the Type

You can explicitly tell Rust what type a variable should be:

```rust
fn main() {
    let x: i32 = 5;           // Explicit: x is a 32-bit integer
    let y: f64 = 3.14;        // Explicit: y is a 64-bit float
    let z: bool = true;       // Explicit: z is a boolean
    let name: &str = "Alice"; // Explicit: name is a string slice
    let c: char = 'A';        // Explicit: c is a character
}
```

### The Syntax

```
let variable_name: Type = value;
                 │
                 └── Colon followed by the type
```

### When Should You Write Type Annotations?

| Situation | Annotate? | Example |
|-----------|-----------|---------|
| Type is obvious from the value | No | `let x = 5;` |
| Rust can't infer the type | Yes | `let v: Vec<i32> = Vec::new();` |
| You want a non-default type | Yes | `let x: u8 = 5;` (instead of default i32) |
| Function parameters | Always | `fn add(a: i32, b: i32)` |
| Function return types | Always | `fn add(a: i32, b: i32) -> i32` |
| For clarity in complex code | Helpful | `let result: HashMap<String, Vec<i32>> = ...` |

> **Real-World Practice:** Most Rust developers omit type annotations when the type is obvious, and add them when it improves readability. The rust-analyzer IDE extension shows you the inferred types inline, so you always know what type a variable has.

---

## Immutability by Default

Here's the big one. **Variables in Rust are immutable by default:**

```rust
fn main() {
    let x = 5;
    println!("x is {}", x);
    
    x = 6;  // ❌ ERROR!
    println!("x is {}", x);
}
```

Compiler error:
```
error[E0384]: cannot assign twice to immutable variable `x`
 --> src/main.rs:4:5
  |
2 |     let x = 5;
  |         - first assignment to `x`
3 |     println!("x is {}", x);
4 |     x = 6;
  |     ^^^^^ cannot assign twice to immutable variable
  |
help: consider making this binding mutable
  |
2 |     let mut x = 5;
  |         +++
```

Notice how the compiler:
1. Tells you WHAT went wrong: "cannot assign twice to immutable variable"
2. Shows you WHERE both assignments happen
3. Suggests HOW to fix it: "consider making this binding mutable: `let mut x`"

### What Does "Immutable" Mean?

```
Immutable = Cannot be changed after creation
Mutable   = Can be changed after creation
```

```
Analogy:

Immutable is like writing with a permanent marker on a whiteboard:
  → You wrote "5" and you CAN'T erase it (without creating a new section)

Mutable is like writing with a dry-erase marker:
  → You wrote "5" and you CAN erase it and write "6"
```

---

## Making Variables Mutable with `mut`

To make a variable changeable, add the `mut` keyword:

```rust
fn main() {
    let mut x = 5;    // ← mut makes it mutable
    println!("x is {}", x);
    
    x = 6;            // ✅ This works now!
    println!("x is {}", x);
    
    x = 7;            // ✅ Can change as many times as you want
    println!("x is {}", x);
}
```

Output:
```
x is 5
x is 6
x is 7
```

### You Can Only Change the Value, Not the Type

```rust
fn main() {
    let mut x = 5;      // x is i32
    x = 10;             // ✅ Fine — still an i32
    // x = "hello";     // ❌ ERROR — can't change from i32 to &str
}
```

Even with `mut`, the **type is fixed** once the variable is created. You can change the value, but not the type.

### When to Use `mut`

| Use Case | Example |
|----------|---------|
| Counters | `let mut count = 0; count += 1;` |
| Accumulators | `let mut sum = 0; for n in numbers { sum += n; }` |
| Buffers | `let mut buffer = String::new(); buffer.push_str("hello");` |
| State that changes | `let mut is_running = true; is_running = false;` |
| Loop variables | `let mut i = 0; while i < 10 { i += 1; }` |

> **Rule of thumb:** Start with `let` (immutable). Only add `mut` when you actually need to change the value. The compiler will tell you if you need `mut` — just follow its suggestions.

---

## Why Immutable by Default?

This is a design decision that confuses people from other languages. Why make variables immutable by default?

### Reason 1: Prevents Accidental Changes

```rust
fn main() {
    let tax_rate = 0.08;     // 8% tax rate
    // ... 200 lines of code ...
    // tax_rate = 0.50;      // ❌ Oops! Would make tax 50%. Compiler catches this.
    
    let total = price * (1.0 + tax_rate);
}
```

Without immutability, you might accidentally change a value somewhere deep in your code and spend hours debugging why your calculations are wrong.

### Reason 2: Makes Code Easier to Understand

```rust
// When you see this:
let name = "Alice";
// ... 100 lines later ...
println!("{}", name);  // You KNOW it's still "Alice". Guaranteed.

// With mut:
let mut name = "Alice";
// ... 100 lines later ...
println!("{}", name);  // Could be anything! Someone might have changed it.
```

Immutability means you can **trust** that a value hasn't changed. Reading `let x = 5` means `x` is 5 *forever* in that scope.

### Reason 3: Enables Compiler Optimizations

When the compiler knows a value won't change, it can:
- Store it in a CPU register instead of memory (faster access)
- Inline the value wherever it's used
- Eliminate unnecessary loads from memory
- Enable auto-vectorization

### Reason 4: Thread Safety

Immutable data is **automatically safe to share across threads** (we'll cover this in Stage 12). If nothing can change a value, multiple threads can read it simultaneously with zero risk of data races.

### The Principle

> In Rust, mutability is an **explicit opt-in**. This means every `mut` in your code is a signal: "Pay attention — this value WILL change." This makes it much easier to understand data flow in complex programs.

---

## Shadowing — Reusing a Variable Name

Here's a concept that's unique to Rust (well, some functional languages have it too). **Shadowing** is when you create a new variable with the same name as an existing one:

```rust
fn main() {
    let x = 5;
    println!("x is {}", x);    // x is 5
    
    let x = x + 1;             // New "x" shadows the old "x"
    println!("x is {}", x);    // x is 6
    
    let x = x * 2;             // Shadows again
    println!("x is {}", x);    // x is 12
}
```

Output:
```
x is 5
x is 6
x is 12
```

### What's Happening?

Each `let x` creates a **brand new variable** that happens to have the same name. The old variable still exists in memory but is no longer accessible by name:

```
Step 1: let x = 5;
┌─────────┐
│ x₁ = 5  │  ← "x" refers to this
└─────────┘

Step 2: let x = x + 1;
┌─────────┐
│ x₁ = 5  │  ← Still exists, but hidden (shadowed)
├─────────┤
│ x₂ = 6  │  ← "x" now refers to this
└─────────┘

Step 3: let x = x * 2;
┌─────────┐
│ x₁ = 5  │  ← Hidden
├─────────┤
│ x₂ = 6  │  ← Hidden
├─────────┤
│ x₃ = 12 │  ← "x" now refers to this
└─────────┘
```

### Shadowing Can Change the Type!

This is a key difference from `mut` — shadowing creates a completely new variable, so it can have a different type:

```rust
fn main() {
    let spaces = "   ";         // &str (a string)
    let spaces = spaces.len();  // usize (a number) — DIFFERENT TYPE!
    println!("Number of spaces: {}", spaces);
}
```

This would NOT work with `mut`:

```rust
fn main() {
    let mut spaces = "   ";     // &str
    // spaces = spaces.len();   // ❌ ERROR: expected &str, got usize
}
```

### When is Shadowing Useful?

#### 1. Converting Types

```rust
fn main() {
    let input = "42";                            // User input: a string
    let input: i32 = input.parse().unwrap();     // Parse to integer, reuse name
    println!("Number: {}", input);               // 42
}
```

Without shadowing, you'd need two names: `input_str` and `input_num`. Shadowing keeps the name clean.

#### 2. Processing Data Step by Step

```rust
fn main() {
    let name = "  Alice  ";          // Raw input
    let name = name.trim();          // Remove whitespace: "Alice"
    let name = name.to_uppercase();  // Convert: "ALICE"
    println!("Hello, {}!", name);
}
```

#### 3. Narrowing from Option

```rust
fn main() {
    let number: Option<i32> = Some(42);
    
    if let Some(number) = number {
        // "number" inside here is i32, not Option<i32>
        println!("The number is {}", number);
    }
}
```

---

## Shadowing vs mut — What's the Difference?

| Feature | `mut` | Shadowing |
|---------|-------|-----------|
| Keyword | `let mut x = 5; x = 6;` | `let x = 5; let x = 6;` |
| Changes value? | ✅ Yes | ✅ Yes (new variable, same name) |
| Changes type? | ❌ No | ✅ Yes |
| Creates new variable? | ❌ No (same memory location) | ✅ Yes (new memory location) |
| Old value accessible? | ❌ Gone (overwritten) | ❌ Gone (shadowed, eventually freed) |
| Compile-time guarantee | Value is mutable | Each binding is still immutable |

### When to Use Which?

```rust
// USE MUT when you need to update the same variable repeatedly
let mut count = 0;
for _ in 0..10 {
    count += 1;  // Modifying the same variable
}

// USE SHADOWING when you're transforming a value into something new
let input = "42";
let input: i32 = input.parse().unwrap();  // Different type, same name
```

---

## Variable Scope — Where Variables Live

A **scope** is a region of code where a variable is valid. In Rust, scope is defined by curly braces `{}`:

```rust
fn main() {                     // ─── main scope starts
    let x = 5;                  // x is valid from here...
    
    {                           // ─── inner scope starts
        let y = 10;             // y is valid from here...
        println!("x: {}, y: {}", x, y);  // ✅ Both accessible
    }                           // ─── inner scope ends. y is DROPPED here.
    
    println!("x: {}", x);      // ✅ x is still valid
    // println!("y: {}", y);   // ❌ ERROR: y is no longer valid!
}                               // ─── main scope ends. x is DROPPED here.
```

### Visualizing Scope

```
fn main() {
│
│   let x = 5;          ← x comes into existence
│   │
│   │   {
│   │   │
│   │   │   let y = 10; ← y comes into existence
│   │   │   │
│   │   │   │            ← x AND y are both valid here
│   │   │   │
│   │   │   ▼            ← y is DROPPED (destroyed) here
│   │   }
│   │
│   │                    ← only x is valid here
│   │
│   ▼                    ← x is DROPPED here
}
```

### What Does "Dropped" Mean?

When a variable goes out of scope (its `}` is reached), Rust **drops** it — meaning:
1. Any cleanup code runs (the `Drop` trait — we'll learn about this in Stage 11)
2. The memory is freed
3. The variable name is no longer valid

This is how Rust manages memory without a garbage collector. **Scope = lifetime**. When the scope ends, the memory is freed. Always. Automatically.

### Shadowing and Scope

Shadowing creates a new variable in the current scope:

```rust
fn main() {
    let x = 5;           // x₁ starts
    let x = 10;          // x₂ starts, x₁ is shadowed
    
    {
        let x = 20;      // x₃ starts (only in this inner scope)
        println!("{}", x); // 20 (x₃)
    }                     // x₃ is dropped
    
    println!("{}", x);   // 10 (back to x₂!)
}
```

---

## Unused Variables

Rust warns you about variables you create but never use:

```rust
fn main() {
    let x = 5;  // ⚠️ Warning: unused variable `x`
}
```

```
warning: unused variable: `x`
 --> src/main.rs:2:9
  |
2 |     let x = 5;
  |         ^ help: if this is intentional, prefix it with an underscore: `_x`
```

### How to Silence the Warning

#### Method 1: Prefix with underscore

```rust
fn main() {
    let _x = 5;  // No warning — underscore means "I know this is unused"
}
```

#### Method 2: Just an underscore (discard the value)

```rust
fn main() {
    let _ = calculate_something();  // Discard the result entirely
}
```

#### Method 3: Use the variable!

The best fix is usually to actually use the variable or remove it if it's not needed.

> **Real-World Practice:** Unused variable warnings help keep your code clean. During development, prefix with `_` to temporarily silence warnings. Before committing code, remove any truly unused variables.

---

## Common Mistakes

### Mistake 1: Forgetting `mut`

```rust
fn main() {
    let count = 0;
    count += 1;  // ❌ cannot assign twice to immutable variable
}
// Fix: let mut count = 0;
```

### Mistake 2: Using a Variable Before Assigning

```rust
fn main() {
    let x: i32;
    println!("{}", x);  // ❌ use of possibly-uninitialized variable
}
```

Rust does NOT give variables a default value (unlike some languages). You MUST initialize before using:

```rust
fn main() {
    let x: i32;
    x = 5;              // ✅ Assign before use
    println!("{}", x);  // ✅ Now it's fine
}
```

### Mistake 3: Trying to Change the Type with `mut`

```rust
fn main() {
    let mut x = 5;
    x = "hello";  // ❌ expected integer, found `&str`
}
// Fix: use shadowing instead: let x = "hello";
```

### Mistake 4: Using camelCase Instead of snake_case

```rust
fn main() {
    let myVariable = 5;  // ⚠️ Warning: should be my_variable
    let firstName = "Alice";  // ⚠️ Warning: should be first_name
}
```

This isn't an error (the code compiles), but the compiler warns you because the Rust convention is `snake_case`.

---

## Exercises

### Exercise 1: Basic Variables

Create variables for your personal info and print them:

```
Name: [your name]
Age: [your age]
Height: [your height in meters]
Is student: [true or false]
Favorite character: [a single character]
```

### Exercise 2: Mutability

Create a mutable counter that starts at 0. Increment it three times, printing after each increment:

```
Count: 1
Count: 2
Count: 3
```

### Exercise 3: Shadowing Chain

Start with the string `"42"`, shadow it to parse into a number, then shadow it again to double it:

```rust
let value = "42";
// Your code here — use shadowing to convert and double
// Final print should show: 84
```

### Exercise 4: Scope Experiment

Predict the output of this code, then run it to check:

```rust
fn main() {
    let x = 1;
    {
        let x = 2;
        {
            let x = 3;
            println!("A: {}", x);
        }
        println!("B: {}", x);
    }
    println!("C: {}", x);
}
```

### Exercise 5: Fix the Errors

Fix each of these programs (they all have exactly one error):

```rust
// Program 1
fn main() {
    let x = 5;
    x = 10;
    println!("{}", x);
}

// Program 2
fn main() {
    let y: i32;
    println!("{}", y);
}

// Program 3
fn main() {
    let mut z = "hello";
    z = 42;
    println!("{}", z);
}
```

---

## Summary

| Concept | Syntax | Description |
|---------|--------|-------------|
| **Create variable** | `let x = 5;` | Immutable by default |
| **Mutable variable** | `let mut x = 5;` | Can change value (not type) |
| **Type annotation** | `let x: i32 = 5;` | Explicitly specify the type |
| **Type inference** | `let x = 5;` | Compiler figures out the type |
| **Shadowing** | `let x = 5; let x = 10;` | New variable, same name (can change type) |
| **Scope** | `{ let x = 5; }` | Variable lives until `}` |
| **Unused var** | `let _x = 5;` | Underscore prefix silences warnings |

### Key Takeaways

1. **`let` creates an immutable binding** — you can't change it
2. **`let mut` creates a mutable binding** — you can change the value (not the type)
3. **Shadowing** creates a brand new variable with the same name (CAN change type)
4. **Scope** determines how long a variable lives — ends at `}`
5. **Rust never gives default values** — you must initialize before use
6. **Immutability by default** prevents bugs and enables optimizations

---

## What's Next?

Now let's look at two special kinds of "variables" that have additional guarantees: constants and statics.

**Next Tutorial:** [Constants & Statics →](./02-constants-and-statics.md)

---

<p align="center">
  <i>Tutorial 1 of 8 — Stage 2: Rust Fundamentals</i>
</p>

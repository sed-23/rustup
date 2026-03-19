# Functions ⚙️

> **Functions are the building blocks of Rust programs. They let you organize code into reusable, named pieces.**

---

## Table of Contents

- [What is a Function?](#what-is-a-function)
- [Defining and Calling Functions](#defining-and-calling-functions)
- [Function Naming Convention](#function-naming-convention)
- [Parameters — Giving Functions Input](#parameters--giving-functions-input)
  - [Multiple Parameters](#multiple-parameters)
  - [Pass by Value vs Pass by Reference](#pass-by-value-vs-pass-by-reference)
- [Return Values — Getting Results Back](#return-values--getting-results-back)
  - [Explicit Return with return](#explicit-return-with-return)
  - [Implicit Return — The Expression Way](#implicit-return--the-expression-way)
  - [Returning Nothing — The Unit Type](#returning-nothing--the-unit-type)
- [Statements vs Expressions — The Core Distinction](#statements-vs-expressions--the-core-distinction)
- [Function Signatures as Documentation](#function-signatures-as-documentation)
- [Functions on the Stack — What Happens When You Call a Function](#functions-on-the-stack--what-happens-when-you-call-a-function)
- [Early Return](#early-return)
- [Nested Functions](#nested-functions)
- [Functions as First-Class Citizens — A Preview](#functions-as-first-class-citizens--a-preview)
- [Diverging Functions — Functions That Never Return](#diverging-functions--functions-that-never-return)
- [Common Patterns](#common-patterns)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What is a Function?

A function is a **named block of code** that:
1. Has a **name** — so you can call it
2. Can accept **inputs** (parameters)
3. Performs some **work**
4. Can produce an **output** (return value)

```
                ┌──────────────┐
  inputs ──────→│   Function   │──────→ output
 (parameters)   │   (code)     │   (return value)
                └──────────────┘
```

Every Rust program starts with one special function:

```rust
fn main() {
    // This is where your program starts
}
```

---

## Defining and Calling Functions

```rust
// DEFINING a function
fn greet() {
    println!("Hello, World!");
}

fn main() {
    // CALLING the function
    greet();   // → Hello, World!
    greet();   // → Hello, World!  (can call as many times as you want)
}
```

### The Anatomy of a Function

```
fn function_name(parameter1: Type1, parameter2: Type2) -> ReturnType {
│  │              │                                       │
│  │              │                                       └── Return type
│  │              └── Parameters (inputs)
│  └── Function name (snake_case)
└── The fn keyword

    // function body
    let result = parameter1 + parameter2;
    result  // ← last expression is returned (no semicolon!)
}
```

### Where Can Functions Be Defined?

Functions can be defined **anywhere** in the file — before or after `main`. Unlike C, there's no need for forward declarations:

```rust
fn main() {
    greet();           // ✅ Works even though greet is defined BELOW
    let sum = add(3, 4);
    println!("Sum: {}", sum);
}

fn greet() {
    println!("Hello!");
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Rust scans the entire file first, so function order doesn't matter.

---

## Function Naming Convention

Functions use **snake_case** — lowercase with underscores:

```rust
// ✅ GOOD — snake_case
fn calculate_area() {}
fn get_user_name() {}
fn is_valid_email() {}
fn print_report() {}

// ❌ BAD — the compiler warns you
fn calculateArea() {}   // ⚠️ camelCase: should be calculate_area
fn GetUserName() {}     // ⚠️ PascalCase: should be get_user_name
fn PRINTREPORT() {}     // ⚠️ should be print_report
```

### Naming Tips

| Pattern | Convention | Example |
|---------|-----------|---------|
| Actions | Start with a verb | `calculate_total`, `send_email`, `validate_input` |
| Getters | Use `get_` prefix | `get_name`, `get_balance` |
| Setters | Use `set_` prefix | `set_name`, `set_balance` |
| Boolean checks | Use `is_`/`has_`/`can_` | `is_empty`, `has_permission`, `can_delete` |
| Conversions | Use `to_`/`from_`/`into_` | `to_string`, `from_bytes`, `into_iter` |
| Constructors | Use `new` | `new`, `with_capacity` |

---

## Parameters — Giving Functions Input

Parameters are variables that a function accepts as input. In Rust, you **must** always specify the type:

```rust
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    greet("Alice");   // Hello, Alice!
    greet("Bob");     // Hello, Bob!
}
```

### The Difference: Parameters vs Arguments

```
fn greet(name: &str) {     ← "name" is a PARAMETER (in the definition)
    println!("Hello, {}!", name);
}

fn main() {
    greet("Alice");          ← "Alice" is an ARGUMENT (in the call)
}
```

- **Parameter**: The variable name in the function definition
- **Argument**: The actual value you pass when calling the function

In practice, most people use these terms interchangeably. That's fine.

### Multiple Parameters

```rust
fn describe_person(name: &str, age: i32, height: f64) {
    println!("{} is {} years old and {:.1}m tall", name, age, height);
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn power(base: f64, exponent: u32) -> f64 {
    base.powi(exponent as i32)
}

fn main() {
    describe_person("Alice", 30, 1.65);
    // Alice is 30 years old and 1.6m tall
    
    println!("3 + 4 = {}", add(3, 4));
    // 3 + 4 = 7
    
    println!("2^10 = {}", power(2.0, 10));
    // 2^10 = 1024
}
```

### Why Must Types Be Specified?

In Rust, function parameter types are **always required**:

```rust
fn add(a: i32, b: i32) -> i32 { a + b }
// NOT: fn add(a, b) { a + b }  // ❌ Types missing!
```

Why?
1. **Clarity**: Anyone reading the function signature knows exactly what types to pass
2. **Safety**: The compiler can check that callers pass the right types
3. **No ambiguity**: The compiler doesn't need to guess (unlike type inference for `let`)

> **Design Philosophy:** Rust requires type annotations at function boundaries (parameters and return types) but infers types within function bodies. This is a deliberate choice — function signatures are the "contract" with the outside world and should be explicit.

### Pass by Value vs Pass by Reference

When you pass a value to a function, you can either give it the **value** (copy/move) or a **reference** (borrow):

```rust
fn print_number(n: i32) {
    //                 ^ Takes ownership (but i32 is Copy, so it's actually copied)
    println!("Number: {}", n);
}

fn print_string(s: String) {
    //                   ^ Takes ownership — the caller LOSES access!
    println!("String: {}", s);
}

fn print_string_ref(s: &String) {
    //                    ^ Borrows a reference — caller keeps ownership
    println!("String: {}", s);
}

fn print_str(s: &str) {
    //            ^ Borrows a string slice — most flexible
    println!("String: {}", s);
}

fn main() {
    let n = 42;
    print_number(n);
    println!("n is still: {}", n);  // ✅ i32 implements Copy
    
    let s = String::from("hello");
    // print_string(s);  // After this, s is MOVED — can't use s anymore!
    // println!("{}", s); // ❌ ERROR: value used after move
    
    let s = String::from("hello");
    print_string_ref(&s);  // ✅ Borrow — s is still valid
    println!("s is still: {}", s);  // ✅ Works!
    
    print_str("hello");           // ✅ String literal
    print_str(&s);                // ✅ String reference
}
```

> **Deep Dive:** Ownership and borrowing are covered in detail in Stage 3. For now, know that:
> - Simple types (`i32`, `f64`, `bool`, `char`) are **copied** automatically
> - Complex types (`String`, `Vec`) are **moved** unless you use `&` to borrow

---

## Return Values — Getting Results Back

Functions can produce a result using the `->` syntax:

```rust
fn add(a: i32, b: i32) -> i32 {
    //                    ^^^^^ Return type
    a + b
    // ^^^^^ The return value (no semicolon = expression returned)
}

fn main() {
    let result = add(3, 4);
    println!("3 + 4 = {}", result);  // 7
}
```

### Explicit Return with `return`

The `return` keyword immediately exits the function with a value:

```rust
fn absolute_value(n: i32) -> i32 {
    if n < 0 {
        return -n;  // Exit early with -n
    }
    n  // This is reached only if n >= 0
}

fn main() {
    println!("|−5| = {}", absolute_value(-5));  // 5
    println!("|3| = {}", absolute_value(3));    // 3
}
```

### Implicit Return — The Expression Way

In Rust, the **last expression** in a function body is automatically returned (if there's no semicolon):

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b     // ← No semicolon = this expression is returned
}

fn multiply(a: i32, b: i32) -> i32 {
    a * b     // ← No semicolon = returned
}

// These are EQUIVALENT:
fn double_explicit(n: i32) -> i32 {
    return n * 2;   // Explicit return
}

fn double_implicit(n: i32) -> i32 {
    n * 2           // Implicit return (idiomatic Rust)
}
```

### ⚠️ The Semicolon Trap

Adding a semicolon changes the behavior dramatically:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b     // ✅ Returns i32
}

fn add_broken(a: i32, b: i32) -> i32 {
    a + b;    // ❌ The semicolon turns this into a statement that returns ()
}
```

The error:
```
error[E0308]: mismatched types
 --> src/main.rs:5:42
  |
5 | fn add_broken(a: i32, b: i32) -> i32 {
  |                                   ^^^ expected `i32`, found `()`
6 |     a + b;
  |          - help: remove this semicolon to return this value
```

**The rule:**
- `a + b` → expression, evaluates to a value (returned)
- `a + b;` → statement, evaluates to `()` (nothing returned)

### Returning Nothing — The Unit Type

Functions that don't return a useful value return `()` (the unit type):

```rust
// These are ALL equivalent:
fn greet() {
    println!("Hello!");
}

fn greet_explicit() -> () {
    println!("Hello!");
}

fn greet_with_return() -> () {
    println!("Hello!");
    return ();
}
```

You almost never write `-> ()` explicitly — just omit the return type.

---

## Statements vs Expressions — The Core Distinction

This is one of the most important concepts in Rust. Almost everything is either a **statement** or an **expression**:

### Statements: Perform Actions, Return Nothing

```rust
fn main() {
    // These are statements:
    let x = 5;            // Variable declaration — returns nothing
    println!("hello");    // Function call with ; — returns nothing
    
    // You CANNOT do this:
    // let y = (let x = 5);  // ❌ ERROR: let is a statement, not an expression
}
```

### Expressions: Produce Values

```rust
fn main() {
    // These are expressions:
    5                     // A literal → produces 5
    5 + 3                 // An arithmetic expression → produces 8
    add(2, 3)             // A function call → produces a value
    true && false         // A boolean expression → produces false
    
    // Blocks {} are expressions!
    let y = {
        let x = 3;
        x + 1       // ← No semicolon = this is the block's value
    };
    println!("y = {}", y);  // y = 4
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

### The Block Expression

A block `{}` is an expression — it evaluates to the last expression inside it:

```rust
fn main() {
    let age = 25;
    
    let category = {
        if age < 13 {
            "child"
        } else if age < 20 {
            "teenager"
        } else {
            "adult"
        }
    };
    // ← category = "adult"
    
    println!("Category: {}", category);
    
    // More complex block expression:
    let result = {
        let a = 10;
        let b = 20;
        let c = a + b;
        c * 2          // ← Value of the block = 60
    };
    
    println!("Result: {}", result);  // 60
}
```

### Quick Reference

| Thing | Statement or Expression? | Produces a Value? |
|-------|-------------------------|-------------------|
| `let x = 5;` | Statement | ❌ |
| `5` | Expression | ✅ `5` |
| `x + 1` | Expression | ✅ (result of addition) |
| `{ let x = 3; x + 1 }` | Expression | ✅ `4` |
| `if condition { a } else { b }` | Expression | ✅ `a` or `b` |
| `fn greet() { ... }` | Statement (item declaration) | ❌ |
| `greet()` | Expression | ✅ `()` |
| `x + 1;` | Statement (expression + `;`) | Discards value |

---

## Function Signatures as Documentation

A well-written function signature tells you everything you need to know about how to use the function:

```rust
fn calculate_bmi(weight_kg: f64, height_m: f64) -> f64 {
    weight_kg / (height_m * height_m)
}
```

Reading this signature tells you:
1. **What it does**: Calculates BMI (from the name)
2. **What it needs**: Weight in kg (f64) and height in meters (f64)
3. **What it returns**: A floating-point number (f64)

Compare this to a function in a dynamically typed language:
```python
def calculate_bmi(weight, height):  # What types? What does it return? 🤷
```

### Type Annotations Act Like Documentation

```rust
// This function signature is a CONTRACT:
fn find_max(numbers: &[i32]) -> Option<i32> {
    // Input: a slice of integers (won't modify them — & means borrow)
    // Output: Option<i32> — might return None if the slice is empty
    if numbers.is_empty() {
        return None;
    }
    let mut max = numbers[0];
    for &n in &numbers[1..] {
        if n > max {
            max = n;
        }
    }
    Some(max)
}
```

---

## Functions on the Stack — What Happens When You Call a Function

When you call a function, the computer creates a **stack frame** for it:

```rust
fn add(a: i32, b: i32) -> i32 {
    let sum = a + b;
    sum
}

fn main() {
    let x = 5;
    let y = 10;
    let result = add(x, y);
    println!("{}", result);
}
```

### Step-by-Step Stack Visualization

```
Step 1: main() starts
┌──────────────────┐
│ main()           │
│   x = 5          │
│   y = 10         │
│   result = ?     │
└──────────────────┘ ← Stack pointer

Step 2: add(5, 10) is called — new frame pushed
┌──────────────────┐
│ add()            │ ← Currently executing
│   a = 5          │
│   b = 10         │
│   sum = ?        │
├──────────────────┤
│ main()           │ ← Waiting for add() to return
│   x = 5          │
│   y = 10         │
│   result = ?     │
└──────────────────┘

Step 3: add() calculates sum = 15
┌──────────────────┐
│ add()            │
│   a = 5          │
│   b = 10         │
│   sum = 15       │
├──────────────────┤
│ main()           │
│   x = 5          │
│   y = 10         │
│   result = ?     │
└──────────────────┘

Step 4: add() returns 15 — its frame is popped
┌──────────────────┐
│ main()           │ ← Resumes execution
│   x = 5          │
│   y = 10         │
│   result = 15    │  ← Return value stored
└──────────────────┘

Step 5: main() finishes — its frame is popped
(stack is empty — program ends)
```

### Key Points About the Stack

1. **Each function call creates a new frame** on top of the stack
2. **Local variables live in the frame** — they're created when the function starts and destroyed when it returns
3. **Stack grows "upward"** (toward higher addresses) and shrinks when functions return
4. **Stack overflow** happens if you have too many nested calls (e.g., infinite recursion):

```rust
fn infinite() {
    infinite();  // Each call adds a frame — eventually the stack is full!
}
// thread 'main' has overflowed its stack
```

---

## Early Return

The `return` keyword exits the function immediately:

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        return Err(String::from("Cannot divide by zero"));
    }
    Ok(a / b)
}

fn find_first_negative(numbers: &[i32]) -> Option<i32> {
    for &n in numbers {
        if n < 0 {
            return Some(n);  // Exit immediately when found
        }
    }
    None  // Only reached if no negative number was found
}

fn main() {
    println!("{:?}", divide(10.0, 3.0));   // Ok(3.3333...)
    println!("{:?}", divide(10.0, 0.0));   // Err("Cannot divide by zero")
    
    let nums = [5, 3, -2, 8, -7];
    println!("{:?}", find_first_negative(&nums));  // Some(-2)
}
```

### Guard Clauses

A common pattern is to put error checks at the top of a function:

```rust
fn process_age(age: i32) -> String {
    // Guard clauses — check error conditions first
    if age < 0 {
        return String::from("Error: age cannot be negative");
    }
    if age > 150 {
        return String::from("Error: age seems unrealistic");
    }
    
    // Main logic (only runs if all guards pass)
    if age < 18 {
        String::from("Minor")
    } else {
        String::from("Adult")
    }
}
```

> **Real-World Practice:** Guard clauses make functions easier to read by handling edge cases first, then focusing on the "happy path" main logic.

---

## Nested Functions

You can define functions inside other functions:

```rust
fn main() {
    fn add(a: i32, b: i32) -> i32 {
        a + b
    }
    
    fn multiply(a: i32, b: i32) -> i32 {
        a * b
    }
    
    println!("3 + 4 = {}", add(3, 4));
    println!("3 × 4 = {}", multiply(3, 4));
}
```

Nested functions:
- Are only accessible within the enclosing function
- **Cannot access** the enclosing function's variables (use closures for that — Stage 8)
- Are useful for helper logic that's only needed in one place

```rust
fn main() {
    let x = 10;
    
    fn helper() {
        // println!("{}", x);  // ❌ ERROR: can't capture environment
    }
    
    // For capturing variables, use closures instead:
    let closure = || println!("{}", x);  // ✅ Closures can capture
    closure();
}
```

---

## Functions as First-Class Citizens — A Preview

In Rust, functions can be passed around like values:

```rust
fn add(a: i32, b: i32) -> i32 { a + b }
fn multiply(a: i32, b: i32) -> i32 { a * b }

fn apply(f: fn(i32, i32) -> i32, a: i32, b: i32) -> i32 {
    f(a, b)
}

fn main() {
    let result1 = apply(add, 3, 4);
    let result2 = apply(multiply, 3, 4);
    
    println!("add(3, 4) = {}", result1);       // 7
    println!("multiply(3, 4) = {}", result2);  // 12
}
```

The type `fn(i32, i32) -> i32` is a **function pointer**. We'll explore this in depth in Stage 8 (Closures).

---

## Diverging Functions — Functions That Never Return

Some functions never return. They use the **never type** `!`:

```rust
fn forever() -> ! {
    loop {
        // Runs forever — never returns
    }
}

fn crash(message: &str) -> ! {
    panic!("Fatal error: {}", message);
    // panic! terminates the program — function never returns normally
}
```

The `!` type is called the "never" type. The most common use is:
- `panic!()` — crashes the program
- `std::process::exit()` — terminates the process
- Infinite loops (`loop { }` with no `break`)

This is useful because `!` can be coerced to any type, which makes code like this work:

```rust
fn main() {
    let value: i32 = match Some(5) {
        Some(n) => n,
        None => panic!("no value!"),  // panic! returns !, which coerces to i32
    };
}
```

---

## Common Patterns

### Pattern 1: Validation Functions

```rust
fn is_valid_email(email: &str) -> bool {
    email.contains('@') && email.contains('.')
}

fn is_valid_age(age: i32) -> bool {
    age >= 0 && age <= 150
}

fn main() {
    println!("valid email: {}", is_valid_email("user@example.com")); // true
    println!("valid email: {}", is_valid_email("not-an-email"));     // false
    println!("valid age: {}", is_valid_age(25));   // true
    println!("valid age: {}", is_valid_age(-5));   // false
}
```

### Pattern 2: Builder-Style Step Functions

```rust
fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 5.0 / 9.0
}

fn main() {
    let boiling_c = 100.0;
    let boiling_f = celsius_to_fahrenheit(boiling_c);
    let back_to_c = fahrenheit_to_celsius(boiling_f);
    
    println!("{:.1}°C = {:.1}°F = {:.1}°C", boiling_c, boiling_f, back_to_c);
    // 100.0°C = 212.0°F = 100.0°C
}
```

### Pattern 3: Functions That Return Multiple Values (Tuple)

```rust
fn divide_with_remainder(dividend: i32, divisor: i32) -> (i32, i32) {
    (dividend / divisor, dividend % divisor)
}

fn swap(a: i32, b: i32) -> (i32, i32) {
    (b, a)
}

fn main() {
    let (quotient, remainder) = divide_with_remainder(17, 5);
    println!("17 / 5 = {} R {}", quotient, remainder);
    
    let (x, y) = swap(10, 20);
    println!("Swapped: {}, {}", x, y);  // 20, 10
}
```

### Pattern 4: Recursive Functions

```rust
fn factorial(n: u64) -> u64 {
    if n <= 1 {
        1
    } else {
        n * factorial(n - 1)
    }
}

fn fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn main() {
    println!("5! = {}", factorial(5));      // 120
    println!("10! = {}", factorial(10));    // 3628800
    
    // Print first 10 Fibonacci numbers
    for i in 0..10 {
        print!("{} ", fibonacci(i));
    }
    println!();
    // 0 1 1 2 3 5 8 13 21 34
}
```

> **Note:** This recursive Fibonacci is O(2^n) — very slow! We'll learn about iterative and memoized approaches in later stages.

### How Recursion Works on the Stack

```rust
fn factorial(n: u64) -> u64 {
    if n <= 1 { 1 } else { n * factorial(n - 1) }
}
// factorial(4):
```

```
Call Stack (factorial(4)):

┌─────────────────┐
│ factorial(1)     │  → returns 1
├─────────────────┤
│ factorial(2)     │  → returns 2 * 1 = 2
├─────────────────┤
│ factorial(3)     │  → returns 3 * 2 = 6
├─────────────────┤
│ factorial(4)     │  → returns 4 * 6 = 24
├─────────────────┤
│ main()           │
└─────────────────┘

Unwinds:
factorial(1) = 1
factorial(2) = 2 × 1 = 2
factorial(3) = 3 × 2 = 6
factorial(4) = 4 × 6 = 24
```

---

## Common Mistakes

### Mistake 1: Missing Return Type

```rust
// fn add(a: i32, b: i32) {  // ❌ No return type — returns ()
//     a + b
// }

fn add(a: i32, b: i32) -> i32 {  // ✅ Specify -> i32
    a + b
}
```

### Mistake 2: Semicolon After Return Expression

```rust
fn double(n: i32) -> i32 {
    // n * 2;    // ❌ Semicolon makes this a statement, returns ()
    n * 2        // ✅ No semicolon — returns the value
}
```

### Mistake 3: Missing Parameter Types

```rust
// fn add(a, b) -> i32 { a + b }  // ❌ Types required!
fn add(a: i32, b: i32) -> i32 { a + b }  // ✅
```

### Mistake 4: Using return Without Semicolon

```rust
fn check(n: i32) -> &'static str {
    if n > 0 {
        return "positive"   // ❌ Missing semicolon after return
    }
    "non-positive"
}

fn check_fixed(n: i32) -> &'static str {
    if n > 0 {
        return "positive";  // ✅ return statements need semicolons
    }
    "non-positive"
}
```

### Mistake 5: Returning a Reference to a Local Variable

```rust
// fn make_greeting() -> &str {
//     let s = String::from("Hello!");
//     &s  // ❌ ERROR: `s` is dropped when the function returns!
//         // The reference would point to freed memory
// }

fn make_greeting() -> String {
    String::from("Hello!")  // ✅ Return the owned String instead
}
```

---

## Exercises

### Exercise 1: Basic Functions

Write three functions:
- `square(n: i32) -> i32` — returns n²
- `cube(n: i32) -> i32` — returns n³
- `is_even(n: i32) -> bool` — returns true if n is even

Test them in `main`.

### Exercise 2: Temperature Converter

Write two functions:
- `celsius_to_fahrenheit(c: f64) -> f64`
- `fahrenheit_to_celsius(f: f64) -> f64`

Convert and print: 0°C, 100°C, 32°F, 212°F.

### Exercise 3: Array Functions

Write these functions:
- `sum(numbers: &[i32]) -> i32` — returns the sum
- `average(numbers: &[i32]) -> f64` — returns the average
- `count_positive(numbers: &[i32]) -> usize` — counts positive numbers

Test with `[−3, 5, −1, 8, 0, 2, −7, 4]`.

### Exercise 4: Recursive Power

Write a recursive function `power(base: i32, exp: u32) -> i32` that calculates base^exp.

Rules: If exp is 0, return 1. Otherwise, return base × power(base, exp − 1).

Test: `power(2, 10)` should return `1024`.

### Exercise 5: FizzBuzz Function

Write a function `fizzbuzz(n: i32) -> String` that returns:
- "FizzBuzz" if n is divisible by both 3 and 5
- "Fizz" if divisible by 3 only
- "Buzz" if divisible by 5 only
- The number as a string otherwise

Call it for 1 through 20.

### Exercise 6: Expression Blocks

Rewrite this using block expressions (no explicit `return`, no `if/else` statements):

```rust
fn classify_number(n: i32) -> &'static str {
    if n > 0 {
        return "positive";
    } else if n < 0 {
        return "negative";
    } else {
        return "zero";
    }
}
```

---

## Summary

| Concept | Syntax | Example |
|---------|--------|---------|
| Define function | `fn name() { }` | `fn greet() { println!("Hi"); }` |
| Parameters | `fn name(param: Type)` | `fn greet(name: &str)` |
| Return type | `fn name() -> Type` | `fn add(a: i32, b: i32) -> i32` |
| Implicit return | Last expression, no `;` | `fn double(n: i32) -> i32 { n * 2 }` |
| Explicit return | `return value;` | `return -1;` |
| No return value | Omit `-> Type` | `fn print() { println!("hi"); }` |
| Unit type | `()` | Functions with no return value return `()` |
| Never type | `!` | `fn crash() -> ! { panic!(); }` |

### Key Takeaways

1. **`fn` declares a function**, `snake_case` naming is mandatory
2. **Parameter types are always required** — no type inference at boundaries
3. **Return type is specified** with `->` (omit for `()`)
4. **The last expression** (without `;`) is the return value — this is idiomatic Rust
5. **Adding `;` turns an expression into a statement** (returns `()` instead!)
6. **Statements** perform actions but produce no value; **expressions** produce values
7. **Block `{}` is an expression** — evaluates to its last expression
8. **Functions create stack frames** — locals live on the stack and are cleaned up on return
9. **Guard clauses** (early returns) make functions easier to read

---

## What's Next?

We can now store data and organize code into functions. Next, let's learn how to make decisions in our code with conditionals!

**Next Tutorial:** [Control Flow — Conditionals →](./06-control-flow-conditionals.md)

---

<p align="center">
  <i>Tutorial 5 of 8 — Stage 2: Rust Fundamentals</i>
</p>

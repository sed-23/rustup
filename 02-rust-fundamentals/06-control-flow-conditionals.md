# Control Flow — Conditionals 🔀

> **Conditionals let your program make decisions. "If this is true, do that. Otherwise, do something else."**

---

## Table of Contents

- [What is Control Flow?](#what-is-control-flow)
- [The if Expression](#the-if-expression)
- [else — The Alternative Path](#else--the-alternative-path)
- [else if — Multiple Conditions](#else-if--multiple-conditions)
- [if as an Expression — Assigning Values](#if-as-an-expression--assigning-values)
- [Conditions Must Be bool](#conditions-must-be-bool)
- [Comparison Operators](#comparison-operators)
- [Logical Operators — Combining Conditions](#logical-operators--combining-conditions)
- [Nested if Statements](#nested-if-statements)
- [match — Rust's Powerful Pattern Matching](#match--rusts-powerful-pattern-matching)
  - [Match with Ranges](#match-with-ranges)
  - [Match with Multiple Patterns](#match-with-multiple-patterns)
  - [Match Must Be Exhaustive](#match-must-be-exhaustive)
  - [Match as an Expression](#match-as-an-expression)
  - [Match Guards](#match-guards)
- [if let — Shorthand for Single-Pattern Match](#if-let--shorthand-for-single-pattern-match)
- [Control Flow in Memory — Branch Prediction](#control-flow-in-memory--branch-prediction)
- [Common Patterns](#common-patterns)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What is Control Flow?

Without control flow, programs run one line at a time from top to bottom. Control flow lets programs **make decisions** and **take different paths**:

```
Without control flow:         With control flow:
┌──────────────┐              ┌──────────────┐
│ Step 1       │              │ Step 1       │
│ Step 2       │              │ Check cond.  │
│ Step 3       │              │   ┌────┴────┐│
│ Step 4       │              │ True    False│
│ Step 5       │              │ Step A  Step B│
└──────────────┘              │   └────┬────┘│
                              │ Step 3       │
                              └──────────────┘
```

Rust has two main conditional constructs:
1. **`if` / `else if` / `else`** — for simple branching
2. **`match`** — for complex pattern matching

---

## The `if` Expression

The simplest form — "if this condition is true, run this code":

```rust
fn main() {
    let temperature = 35;
    
    if temperature > 30 {
        println!("It's hot outside! 🔥");
    }
    
    println!("Program continues...");
}
```

Output:
```
It's hot outside! 🔥
Program continues...
```

### The Syntax

```
if condition {
    // code to run when condition is true
}
```

```
if temperature > 30 {
│  │                │
│  │                └── Opening brace (REQUIRED — no single-line ifs!)
│  └── Condition (must evaluate to bool)
└── The keyword

    println!("It's hot!");

}  ← Closing brace
```

### No Parentheses Around the Condition!

Unlike C, Java, and JavaScript, Rust does **not** use parentheses around the condition:

```rust
// ✅ Rust way — no parentheses
if temperature > 30 {
    println!("Hot!");
}

// ⚠️ This works but the compiler warns: "unnecessary parentheses"
if (temperature > 30) {
    println!("Hot!");
}

// ❌ In C/Java/JS you'd write:
// if (temperature > 30) { ... }
```

### Braces Are ALWAYS Required

In Rust, you must always use `{}` even for single-line bodies:

```rust
// ❌ This does NOT work in Rust
// if temperature > 30
//     println!("Hot!");

// ✅ Always use braces
if temperature > 30 {
    println!("Hot!");
}
```

This prevents a class of bugs common in C where adding a second line accidentally falls outside the if:

```c
// C bug (without braces):
if (logged_in)
    show_dashboard();
    show_admin_panel();  // ← This ALWAYS runs! Not inside the if!
```

In Rust, this is impossible because braces are required.

---

## `else` — The Alternative Path

`else` runs when the `if` condition is false:

```rust
fn main() {
    let age = 15;
    
    if age >= 18 {
        println!("You can vote!");
    } else {
        println!("Too young to vote.");
    }
}
```

Output:
```
Too young to vote.
```

### Flow Diagram

```
        ┌─────────────┐
        │ age >= 18?   │
        └─────┬───────┘
         ┌────┴────┐
        Yes       No
         │         │
   ┌─────┴─────┐ ┌─┴──────────┐
   │ "Can vote" │ │ "Too young"│
   └─────┬─────┘ └─┬──────────┘
         └────┬────┘
              ▼
        (continue)
```

---

## `else if` — Multiple Conditions

Chain multiple conditions together:

```rust
fn main() {
    let score = 85;
    
    if score >= 90 {
        println!("Grade: A");
    } else if score >= 80 {
        println!("Grade: B");
    } else if score >= 70 {
        println!("Grade: C");
    } else if score >= 60 {
        println!("Grade: D");
    } else {
        println!("Grade: F");
    }
}
```

Output:
```
Grade: B
```

### Evaluation Order Matters!

Conditions are checked **top to bottom**. The **first** matching condition wins:

```rust
fn main() {
    let score = 95;
    
    // ❌ WRONG ORDER — score >= 60 matches first!
    if score >= 60 {
        println!("Grade: D");  // This prints for score 95!
    } else if score >= 70 {
        println!("Grade: C");  // Never reached for 95
    } else if score >= 80 {
        println!("Grade: B");  // Never reached for 95
    } else if score >= 90 {
        println!("Grade: A");  // Never reached for 95
    }
    
    // ✅ CORRECT ORDER — most specific first
    if score >= 90 {
        println!("Grade: A");  // This is what we want
    } else if score >= 80 {
        println!("Grade: B");
    } else if score >= 70 {
        println!("Grade: C");
    } else if score >= 60 {
        println!("Grade: D");
    } else {
        println!("Grade: F");
    }
}
```

> **Rule of thumb:** When using `else if`, put the **most restrictive** condition first.

### When to Use `match` Instead

If you have more than 3-4 `else if` branches, consider using `match` instead — it's cleaner and the compiler can optimize it better.

---

## `if` as an Expression — Assigning Values

In Rust, `if` is an **expression**, not just a statement. This means it produces a value:

```rust
fn main() {
    let age = 20;
    
    // if/else as an expression — assigns a value
    let category = if age >= 18 { "adult" } else { "minor" };
    
    println!("{} is an {}", age, category);
    // 20 is an adult
}
```

### This Replaces the Ternary Operator

Many languages have a ternary operator: `condition ? value1 : value2`.
Rust doesn't! Instead, `if/else` IS the ternary:

```
C/JavaScript: let x = age >= 18 ? "adult" : "minor";
Rust:         let x = if age >= 18 { "adult" } else { "minor" };
```

### Rules for `if` as an Expression

1. **Both branches must return the same type:**

```rust
let x = if true { 5 } else { 10 };       // ✅ Both i32
// let y = if true { 5 } else { "hello" }; // ❌ i32 vs &str — mismatch!
```

2. **The `else` is required** (otherwise, what's the value when the condition is false?):

```rust
// let x = if true { 5 };  // ❌ ERROR: if without else isn't a complete expression
let x = if true { 5 } else { 0 };  // ✅
```

3. **No semicolons on the value expressions** inside the blocks:

```rust
let x = if true {
    5          // ✅ No semicolon — this is the returned value
} else {
    10         // ✅ No semicolon
};             // ← Semicolon here (end of the let statement)
```

### Multi-Line `if` Expression

```rust
fn main() {
    let temperature = 25;
    
    let description = if temperature > 35 {
        "scorching"
    } else if temperature > 25 {
        "warm"
    } else if temperature > 15 {
        "mild"
    } else if temperature > 5 {
        "cool"
    } else {
        "freezing"
    };
    
    println!("It's {} outside ({} degrees)", description, temperature);
    // It's mild outside (25 degrees)
}
```

---

## Conditions Must Be `bool`

Unlike some languages, Rust does NOT implicitly convert values to bool:

```rust
fn main() {
    let x = 5;
    
    // ❌ WRONG — Rust won't treat a number as a boolean
    // if x {
    //     println!("truthy");
    // }
    // Error: expected `bool`, found `{integer}`
    
    // ✅ Be explicit about what you're checking
    if x != 0 {
        println!("x is non-zero");
    }
    
    if x > 0 {
        println!("x is positive");
    }
    
    // Same for strings:
    let name = "";
    // if name { }         // ❌ Can't use &str as a bool
    if !name.is_empty() {  // ✅ Explicit check
        println!("Name is: {}", name);
    }
}
```

In other languages:
```
JavaScript:  if (0) → false,  if ("") → false,  if (null) → false  (implicit truthiness)
Python:      if 0: → false,   if "": → false,   if None: → false   (implicit truthiness)
Rust:        if 0 → COMPILE ERROR!   (no implicit conversion)
```

This is safer because it prevents bugs like `if (x = 5)` (assignment instead of comparison) which is valid in C but almost always a mistake.

---

## Comparison Operators

| Operator | Meaning | Example | Result |
|----------|---------|---------|--------|
| `==` | Equal to | `5 == 5` | `true` |
| `!=` | Not equal to | `5 != 3` | `true` |
| `<` | Less than | `3 < 5` | `true` |
| `>` | Greater than | `5 > 3` | `true` |
| `<=` | Less than or equal | `5 <= 5` | `true` |
| `>=` | Greater than or equal | `5 >= 3` | `true` |

```rust
fn main() {
    let a = 10;
    let b = 20;
    
    println!("{} == {}: {}", a, b, a == b);  // false
    println!("{} != {}: {}", a, b, a != b);  // true
    println!("{} <  {}: {}", a, b, a < b);   // true
    println!("{} >  {}: {}", a, b, a > b);   // false
    println!("{} <= {}: {}", a, b, a <= b);   // true
    println!("{} >= {}: {}", a, b, a >= b);   // false
}
```

### Important: Can Only Compare Same Types

```rust
fn main() {
    // let result = 5 == 5.0;  // ❌ ERROR: can't compare i32 with f64
    let result = 5.0 == 5.0;   // ✅ Same type
    let result = 5 == 5;       // ✅ Same type
    let result = (5 as f64) == 5.0;  // ✅ Convert first
}
```

---

## Logical Operators — Combining Conditions

| Operator | Name | Meaning | Example |
|----------|------|---------|---------|
| `&&` | AND | Both must be true | `a && b` |
| `\|\|` | OR | At least one must be true | `a \|\| b` |
| `!` | NOT | Flips true/false | `!a` |

```rust
fn main() {
    let age = 25;
    let has_id = true;
    let is_vip = false;
    
    // AND: both conditions must be true
    if age >= 18 && has_id {
        println!("Access granted: adult with ID");
    }
    
    // OR: at least one must be true
    if is_vip || age >= 21 {
        println!("Welcome to the VIP area");
    }
    
    // NOT: inverts the condition
    if !is_vip {
        println!("Not a VIP member");
    }
    
    // Combining all three
    if (age >= 18 && has_id) || is_vip {
        println!("Can enter the venue");
    }
}
```

### Operator Precedence

```
Highest priority first:
1. !    (NOT)
2. ==, !=, <, >, <=, >=  (comparisons)
3. &&   (AND)
4. ||   (OR)
```

```rust
fn main() {
    let a = true;
    let b = false;
    let c = true;
    
    // This:
    let result = a || b && c;
    // Is evaluated as:
    let result = a || (b && c);  // Because && has higher precedence
    // = true || (false && true)
    // = true || false
    // = true
    
    // Use parentheses to be explicit:
    let result = (a || b) && c;
    // = (true || false) && true
    // = true && true
    // = true
}
```

> **Real-World Practice:** Always use parentheses when mixing `&&` and `||` to make your intent clear. Don't rely on precedence rules — other developers (and future you) will thank you.

### Short-Circuit Evaluation (Reminder)

`&&` and `||` stop evaluating as soon as the result is determined:

```rust
fn main() {
    let list: Vec<i32> = vec![];
    
    // ✅ Safe: if list is empty, list[0] is NEVER accessed
    if !list.is_empty() && list[0] > 5 {
        println!("First element is > 5");
    }
    
    // ❌ DANGEROUS: always evaluates list[0] — crashes on empty list!
    // if list[0] > 5 && !list.is_empty() { }
}
```

---

## Nested `if` Statements

You can put `if` inside `if`:

```rust
fn main() {
    let age = 25;
    let has_ticket = true;
    let is_vip = false;
    
    if age >= 18 {
        println!("Age check passed.");
        
        if has_ticket {
            println!("Ticket verified.");
            
            if is_vip {
                println!("Welcome to the VIP section! 🌟");
            } else {
                println!("Welcome to general admission!");
            }
        } else {
            println!("Sorry, you need a ticket.");
        }
    } else {
        println!("Sorry, must be 18 or older.");
    }
}
```

### Avoid Deep Nesting — Use Guard Clauses

Deeply nested `if` statements are hard to read. Flatten them with early returns:

```rust
// ❌ Deep nesting — hard to read
fn check_access(age: i32, has_ticket: bool, is_banned: bool) -> &'static str {
    if !is_banned {
        if age >= 18 {
            if has_ticket {
                "Welcome!"
            } else {
                "Need a ticket"
            }
        } else {
            "Too young"
        }
    } else {
        "Banned"
    }
}

// ✅ Flat — guard clauses with early return
fn check_access_flat(age: i32, has_ticket: bool, is_banned: bool) -> &'static str {
    if is_banned {
        return "Banned";
    }
    if age < 18 {
        return "Too young";
    }
    if !has_ticket {
        return "Need a ticket";
    }
    "Welcome!"
}
```

The flat version is much easier to follow: handle errors first, then the happy path.

---

## `match` — Rust's Powerful Pattern Matching

`match` is like a super-powered `switch` statement. It compares a value against patterns:

```rust
fn main() {
    let number = 3;
    
    match number {
        1 => println!("One"),
        2 => println!("Two"),
        3 => println!("Three"),
        4 => println!("Four"),
        5 => println!("Five"),
        _ => println!("Something else"),  // _ = default/catch-all
    }
    // Output: Three
}
```

### The Syntax

```
match value {
    pattern1 => expression1,
    pattern2 => expression2,
    pattern3 => {
        // Multi-line block
        expression3
    },
    _ => default_expression,
}
```

- Each line is called an **arm**
- `=>` separates the pattern from the code to execute
- `_` is the **wildcard** pattern — matches anything
- Arms are checked top to bottom — first match wins

### Multi-Line Arms

Use `{}` for arms with multiple statements:

```rust
fn main() {
    let day = "Monday";
    
    match day {
        "Monday" => {
            println!("Start of the work week");
            println!("Time to be productive!");
        },
        "Friday" => {
            println!("Almost weekend!");
            println!("TGIF! 🎉");
        },
        "Saturday" | "Sunday" => println!("Weekend! 🎊"),
        _ => println!("Regular weekday"),
    }
}
```

### Match with Ranges

```rust
fn main() {
    let score = 85;
    
    let grade = match score {
        90..=100 => "A",   // 90 to 100 (inclusive)
        80..=89  => "B",   // 80 to 89
        70..=79  => "C",   // 70 to 79
        60..=69  => "D",   // 60 to 69
        0..=59   => "F",   // 0 to 59
        _ => "Invalid",    // Everything else (negative numbers, > 100)
    };
    
    println!("Score: {} → Grade: {}", score, grade);
    // Score: 85 → Grade: B
}
```

The `..=` is the **inclusive range** operator:
- `1..=5` means 1, 2, 3, 4, 5 (includes both endpoints)
- `1..5` means 1, 2, 3, 4 (excludes the end — used in `for` loops, NOT in `match`)

### Match with Multiple Patterns

Use `|` (pipe) to match multiple values in one arm:

```rust
fn main() {
    let day = "Wednesday";
    
    let day_type = match day {
        "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday" => "Weekday",
        "Saturday" | "Sunday" => "Weekend",
        _ => "Unknown",
    };
    
    println!("{} is a {}", day, day_type);
    // Wednesday is a Weekday
}
```

### Match Must Be Exhaustive

The compiler **requires** that you handle ALL possible values. This prevents bugs:

```rust
fn main() {
    let coin = "penny";
    
    // ❌ ERROR: non-exhaustive patterns (what if coin is "dime"?)
    // let value = match coin {
    //     "penny" => 1,
    //     "nickel" => 5,
    // };
    
    // ✅ Handle all cases with a wildcard
    let value = match coin {
        "penny" => 1,
        "nickel" => 5,
        "dime" => 10,
        "quarter" => 25,
        _ => 0,  // Unknown coin
    };
    
    println!("{} = {} cents", coin, value);
}
```

For numeric types, `_` is essential because there are billions of possible values:

```rust
let x: i32 = 42;
match x {
    0 => println!("zero"),
    1..=100 => println!("between 1 and 100"),
    _ => println!("larger than 100 or negative"),
    // Without _, the compiler would complain about unhandled values
}
```

### Match as an Expression

Like `if`, `match` is an expression that produces a value:

```rust
fn main() {
    let number = 7;
    
    let description = match number % 2 {
        0 => "even",
        1 => "odd",
        _ => unreachable!(),  // This can never happen for % 2
    };
    
    println!("{} is {}", number, description);
    // 7 is odd
}
```

### Match Guards

Add an extra condition with `if` after a pattern:

```rust
fn main() {
    let number = 4;
    
    match number {
        n if n < 0 => println!("{} is negative", n),
        0 => println!("zero"),
        n if n % 2 == 0 => println!("{} is positive and even", n),
        n => println!("{} is positive and odd", n),
    }
    // 4 is positive and even
}
```

### Binding Values in Match Arms

You can bind the matched value to a variable:

```rust
fn main() {
    let age = 25;
    
    match age {
        0 => println!("Just born!"),
        1..=12 => println!("Child aged {}", age),
        13..=17 => println!("Teenager aged {}", age),
        n @ 18..=64 => println!("Adult aged {} (working years)", n),
        //^--- bind the matched value to `n`
        n @ 65.. => println!("Senior aged {} (retirement)", n),
        _ => unreachable!(),
    }
    // Adult aged 25 (working years)
}
```

The `@` syntax binds the matched value while also checking it against the pattern.

---

## `if let` — Shorthand for Single-Pattern Match

When you only care about ONE pattern and want to ignore everything else:

```rust
fn main() {
    let some_value: Option<i32> = Some(42);
    
    // Full match — verbose for just checking one pattern
    match some_value {
        Some(value) => println!("Got: {}", value),
        None => {},  // Do nothing for None
    }
    
    // if let — much cleaner for the same thing
    if let Some(value) = some_value {
        println!("Got: {}", value);
    }
    
    // You can add else for the other case:
    if let Some(value) = some_value {
        println!("Got: {}", value);
    } else {
        println!("Got nothing!");
    }
}
```

### When to Use `if let` vs `match`

```rust
// Use if let when you care about ONE pattern:
if let Some(x) = maybe_value {
    println!("Found: {}", x);
}

// Use match when you care about MULTIPLE patterns:
match maybe_value {
    Some(x) if x > 0 => println!("Positive: {}", x),
    Some(x) => println!("Non-positive: {}", x),
    None => println!("Nothing"),
}
```

---

## Control Flow in Memory — Branch Prediction

When Rust compiles an `if/else`, the CPU must decide which branch to take. Modern CPUs use **branch prediction** to guess which way an `if` will go:

```
if condition {  
    Branch A (predicted path)     ← CPU starts executing this SPECULATIVELY
} else {
    Branch B
}
```

If the prediction is wrong, the CPU has to discard its speculative work (**branch misprediction penalty** — typically 10-20 CPU cycles). This is why:

1. **`match` can be faster than long `else if` chains** — the compiler can generate a jump table
2. **Frequently-true conditions should come first** in `else if` chains
3. **Branchless code** can sometimes be faster for simple conditionals

```rust
// Branchless: avoids branch prediction entirely
fn abs_branchless(x: i32) -> i32 {
    let mask = x >> 31;  // All 1s if negative, all 0s if positive
    (x ^ mask) - mask    // Two's complement trick
}

// Normal: uses a branch
fn abs_normal(x: i32) -> i32 {
    if x < 0 { -x } else { x }
}
```

> **Real-World Practice:** Don't optimize branches prematurely. The compiler and CPU are very good at this. Focus on writing clear, readable code. Profile first, optimize later.

---

## Common Patterns

### Pattern 1: Clamping a Value

```rust
fn clamp(value: i32, min: i32, max: i32) -> i32 {
    if value < min {
        min
    } else if value > max {
        max
    } else {
        value
    }
}

fn main() {
    println!("{}", clamp(150, 0, 100));  // 100
    println!("{}", clamp(-20, 0, 100));  // 0
    println!("{}", clamp(50, 0, 100));   // 50
}
```

### Pattern 2: Categorizing Input

```rust
fn classify_char(c: char) -> &'static str {
    match c {
        'a'..='z' => "lowercase letter",
        'A'..='Z' => "uppercase letter",
        '0'..='9' => "digit",
        ' ' | '\t' | '\n' => "whitespace",
        _ => "other",
    }
}

fn main() {
    for c in ['A', 'z', '5', ' ', '!'] {
        println!("'{}' is a {}", c, classify_char(c));
    }
}
```

### Pattern 3: Mapping Day to Number

```rust
fn day_number(day: &str) -> Option<u8> {
    match day.to_lowercase().as_str() {
        "monday"    => Some(1),
        "tuesday"   => Some(2),
        "wednesday" => Some(3),
        "thursday"  => Some(4),
        "friday"    => Some(5),
        "saturday"  => Some(6),
        "sunday"    => Some(7),
        _ => None,
    }
}

fn main() {
    if let Some(num) = day_number("Wednesday") {
        println!("Wednesday is day #{}", num);  // day #3
    }
    
    if let None = day_number("Funday") {
        println!("Funday is not a real day!");
    }
}
```

### Pattern 4: FizzBuzz with Match

```rust
fn fizzbuzz(n: i32) -> String {
    match (n % 3, n % 5) {
        (0, 0) => String::from("FizzBuzz"),
        (0, _) => String::from("Fizz"),
        (_, 0) => String::from("Buzz"),
        _ => n.to_string(),
    }
}

fn main() {
    for i in 1..=20 {
        println!("{:>2}: {}", i, fizzbuzz(i));
    }
}
```

This demonstrates matching on a **tuple** — we match both `n % 3` and `n % 5` at once.

---

## Common Mistakes

### Mistake 1: Non-bool Condition

```rust
let x = 5;
// if x { }       // ❌ ERROR: expected bool, found integer
if x != 0 { }     // ✅ Explicit comparison
```

### Mistake 2: Mismatched Types in if Expression

```rust
// let val = if true { 5 } else { "five" };  // ❌ i32 vs &str
let val = if true { 5 } else { 10 };         // ✅ Both i32
```

### Mistake 3: Missing else in if Expression

```rust
// let val = if true { 5 };  // ❌ What's the value when false?
let val = if true { 5 } else { 0 };  // ✅
```

### Mistake 4: Non-exhaustive match

```rust
let x: i32 = 5;
// match x {         // ❌ Not all cases handled
//     1 => "one",
//     2 => "two",
// }
match x {
    1 => println!("one"),
    2 => println!("two"),
    _ => println!("other"),  // ✅ Wildcard catches everything else
}
```

### Mistake 5: Semicolons in match Arms When Used as Expression

```rust
// let grade = match score {
//     90..=100 => { "A"; },  // ❌ Returns (), not &str
//     _ => { "F"; },
// };

let grade = match score {
    90..=100 => "A",   // ✅ No semicolon — returns the value
    _ => "F",
};
```

### Mistake 6: Using = Instead of == in Conditions

```rust
let x = 5;
// if x = 5 { }   // ❌ ERROR: this is assignment, not comparison
if x == 5 { }     // ✅ Double equals for comparison
```

Rust actually catches this — in C, `if (x = 5)` would silently assign 5 to x (a common bug). Rust prevents it entirely.

---

## Exercises

### Exercise 1: Number Classifier

Write a function that takes an `i32` and prints:
- "positive" if > 0
- "negative" if < 0
- "zero" if == 0

Additionally, if the number is positive, also print whether it's even or odd.

### Exercise 2: Grading System

Write a function `grade(score: u32) -> &'static str` using `match` with ranges that returns:
- "A+" for 97-100
- "A" for 93-96
- "A-" for 90-92
- "B+" for 87-89
- "B" for 83-86
- "B-" for 80-82
- "C" for 70-79
- "D" for 60-69
- "F" for 0-59
- "Invalid" for > 100

### Exercise 3: Season from Month

Write a function that takes a month number (1-12) and returns the season:
- 12, 1, 2 → "Winter"
- 3, 4, 5 → "Spring"
- 6, 7, 8 → "Summer"
- 9, 10, 11 → "Autumn"

Use `match` with the `|` operator.

### Exercise 4: Calculator

Write a function `calculate(a: f64, op: char, b: f64) -> Option<f64>` that:
- Adds if `op` is `'+'`
- Subtracts if `op` is `'-'`
- Multiplies if `op` is `'*'`
- Divides if `op` is `'/'` (return `None` if b is 0)
- Returns `None` for unknown operators

### Exercise 5: Leap Year

Write a function `is_leap_year(year: u32) -> bool` using:
- Divisible by 4 → leap year
- BUT divisible by 100 → NOT a leap year
- BUT divisible by 400 → IS a leap year

Test with: 2000 (true), 1900 (false), 2024 (true), 2023 (false).

### Exercise 6: Rock Paper Scissors

Write a `match` expression that determines the winner of Rock Paper Scissors. Take two `&str` inputs ("rock", "paper", "scissors") and return "Player 1 wins", "Player 2 wins", or "Tie".

Hint: Match on a tuple of both inputs: `match (p1, p2) { ... }`.

---

## Summary

### `if` / `else if` / `else`

| Feature | Syntax |
|---------|--------|
| Basic if | `if condition { code }` |
| If-else | `if cond { A } else { B }` |
| Else-if chain | `if c1 { } else if c2 { } else { }` |
| As expression | `let x = if cond { A } else { B };` |

### `match`

| Feature | Syntax |
|---------|--------|
| Basic match | `match value { pattern => code, _ => default }` |
| Multiple patterns | `1 \| 2 \| 3 => code` |
| Range | `1..=10 => code` |
| Guard | `n if n > 0 => code` |
| Binding | `n @ 1..=10 => code` |
| As expression | `let x = match value { ... };` |

### `if let`

| Feature | Syntax |
|---------|--------|
| Basic | `if let Pattern = value { code }` |
| With else | `if let Pattern = value { A } else { B }` |

### Key Takeaways

1. **No parentheses** around conditions, **braces always required**
2. **Conditions must be `bool`** — no truthy/falsy conversions
3. **`if` is an expression** — can be used on the right side of `let`
4. **`match` must be exhaustive** — all possible values must be handled
5. **`match` is preferred** over long `else if` chains — cleaner and more powerful
6. **`if let`** is shorthand for matching a single pattern
7. **Use guard clauses** (early return) instead of deep nesting
8. **Short-circuit evaluation** (`&&`, `||`) can prevent errors

---

## What's Next?

We can now make decisions. Next, let's learn how to repeat actions with loops!

**Next Tutorial:** [Control Flow — Loops →](./07-control-flow-loops.md)

---

<p align="center">
  <i>Tutorial 6 of 8 — Stage 2: Rust Fundamentals</i>
</p>

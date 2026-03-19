# Comments & Code Style 📝

> **Good code explains itself. Great code is documented. Comments and consistent style make your code readable, maintainable, and professional.**

---

## Table of Contents

- [Why Comments and Code Style Matter](#why-comments-and-code-style-matter)
- [Line Comments](#line-comments)
- [Block Comments](#block-comments)
- [Doc Comments](#doc-comments)
  - [Outer Doc Comments (///)](#outer-doc-comments-)
  - [Inner Doc Comments (//!)](#inner-doc-comments-)
  - [Markdown in Doc Comments](#markdown-in-doc-comments)
  - [Code Examples in Doc Comments](#code-examples-in-doc-comments)
  - [Common Doc Sections](#common-doc-sections)
- [Generating Documentation with cargo doc](#generating-documentation-with-cargo-doc)
- [Rust Naming Conventions](#rust-naming-conventions)
- [Code Formatting with rustfmt](#code-formatting-with-rustfmt)
- [Indentation and Braces](#indentation-and-braces)
- [Line Length](#line-length)
- [Idiomatic Rust Style](#idiomatic-rust-style)
- [When to Comment and When Not To](#when-to-comment-and-when-not-to)
- [Real-World Code Style Examples](#real-world-code-style-examples)
- [Clippy — The Rust Linter](#clippy--the-rust-linter)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Why Comments and Code Style Matter

You write code once, but it gets **read** dozens or hundreds of times — by colleagues, by future you, by open source contributors. Consistent style and clear comments dramatically reduce the time it takes to understand code.

```
Time spent on code:
├── Writing:     20%
└── Reading:     80%  ← Comments and style help HERE
```

Rust goes further than most languages by providing:
1. **`rustfmt`** — automatic code formatter (standard style, no debates)
2. **`clippy`** — linter that catches common mistakes and suggests improvements
3. **Doc comments** — first-class documentation that compiles to HTML

---

## Line Comments

The most common comment type. Everything after `//` on that line is ignored by the compiler:

```rust
fn main() {
    // This is a line comment
    let x = 5;  // This is an inline comment
    
    // You can stack multiple line comments
    // to describe something that needs
    // a longer explanation
    let result = x * 42;
    
    println!("{}", result);
}
```

### Rules for Line Comments

```rust
// ✅ Comments start with // followed by a space (convention)
//✅ Technically works without a space, but looks cramped
// ✅ A comment on its own line
let x = 5; // ✅ An inline comment (after code)

// ❌ Don't put comments where they're useless:
let y = 10; // set y to 10  ← This adds NOTHING
```

### When to Use Line Comments

```rust
// ✅ GOOD: Explain WHY, not WHAT
let timeout = 30; // Server requires response within 30s

// ❌ BAD: Explains what the code obviously does
let x = 5; // Assign 5 to x

// ✅ GOOD: Clarify non-obvious behavior
let mask = 0xFF; // Extract the lowest 8 bits

// ✅ GOOD: Warn about edge cases
// Note: This panics if the input array is empty
let first = &arr[0];
```

---

## Block Comments

Rust supports C-style block comments with `/* ... */`:

```rust
fn main() {
    /* This is a block comment.
       It can span multiple lines. */
    
    let x = /* you can even put them inline */ 5;
    
    /*
     * Some people use this style for multi-line
     * block comments, with asterisks on each line.
     * This is valid but less common in Rust.
     */
}
```

### Block Comments Can Nest

Unlike C/C++, Rust's block comments can be **nested**, which is very helpful when commenting out code that already contains comments:

```rust
fn main() {
    /* 
    This outer comment contains:
        /* This inner comment */
    And it still works!
    */
    
    println!("Nested block comments work in Rust!");
}
```

In C/C++, the inner `*/` would end the outer comment prematurely — but Rust handles nesting correctly.

### When to Use Block Comments

Block comments are relatively **rare** in Rust code. Most Rustaceans prefer stacked `//` comments:

```rust
// ✅ Preferred in Rust (idiomatic)
// This function calculates the area
// of a rectangle given width and height.

/* ⚠ Less common, but valid */
/* This function calculates the area
   of a rectangle given width and height. */
```

> **Convention:** Use `//` for most comments. Reserve `/* */` for temporarily commenting out blocks of code during debugging.

---

## Doc Comments

Doc comments are **special comments** that generate HTML documentation. They're a core part of Rust's ecosystem — every library on [crates.io](https://crates.io) uses them.

### Outer Doc Comments (`///`)

Used to document the **item that follows** — functions, structs, enums, modules, etc.:

```rust
/// Calculates the area of a rectangle.
///
/// Takes the width and height as parameters and returns
/// their product. Both values must be non-negative.
///
/// # Arguments
///
/// * `width` - The width of the rectangle
/// * `height` - The height of the rectangle
///
/// # Returns
///
/// The area as a `f64` value.
///
/// # Examples
///
/// ```
/// let area = calculate_area(5.0, 3.0);
/// assert_eq!(area, 15.0);
/// ```
fn calculate_area(width: f64, height: f64) -> f64 {
    width * height
}
```

### Inner Doc Comments (`//!`)

Used to document the **enclosing item** — typically placed at the top of a file to document the module or crate:

```rust
//! # My Math Library
//!
//! `my_math_lib` provides basic mathematical operations
//! for common geometric calculations.
//!
//! ## Features
//!
//! - Area calculations for common shapes
//! - Perimeter calculations
//! - Volume calculations for 3D shapes
//!
//! ## Quick Start
//!
//! ```
//! use my_math_lib::calculate_area;
//! let area = calculate_area(5.0, 3.0);
//! ```

/// Calculates the area of a rectangle.
fn calculate_area(width: f64, height: f64) -> f64 {
    width * height
}
```

### Outer vs Inner Doc Comments

```
//! Documents the thing I'm INSIDE of (module/crate)
/// Documents the thing BELOW me (function/struct/enum)
```

```rust
//! This documents the module (the file itself)

/// This documents the function below
fn my_function() {}

/// This documents the struct below
struct MyStruct {}

/// This documents the enum below
enum MyEnum {}
```

### Markdown in Doc Comments

Doc comments support full **Markdown** formatting:

```rust
/// # Heading 1
/// ## Heading 2
///
/// This has **bold**, *italic*, and `inline code`.
///
/// Here's a list:
/// - Item one
/// - Item two
/// - Item three
///
/// And a numbered list:
/// 1. First
/// 2. Second
/// 3. Third
///
/// Link: [Rust Book](https://doc.rust-lang.org/book/)
///
/// Table:
///
/// | Shape     | Formula     |
/// |-----------|-------------|
/// | Rectangle | w × h       |
/// | Circle    | π × r²     |
/// | Triangle  | ½ × b × h  |
fn documented_function() {}
```

### Code Examples in Doc Comments

Code blocks in doc comments are **tested** by `cargo test`! This ensures your examples never go out of date:

```rust
/// Doubles the input value.
///
/// # Examples
///
/// ```
/// let result = double(5);
/// assert_eq!(result, 10);
/// ```
///
/// Works with negative numbers too:
///
/// ```
/// let result = double(-3);
/// assert_eq!(result, -6);
/// ```
fn double(x: i32) -> i32 {
    x * 2
}
```

Running `cargo test` will actually compile and run those examples!

### Hiding Lines in Doc Examples

Use `#` to hide setup lines that aren't relevant to the reader:

```rust
/// Calculates the average of a slice.
///
/// # Examples
///
/// ```
/// # // This line is hidden from the rendered docs
/// # fn main() {
/// let numbers = [1.0, 2.0, 3.0, 4.0, 5.0];
/// let avg = average(&numbers);
/// assert_eq!(avg, 3.0);
/// # }
/// # fn average(nums: &[f64]) -> f64 {
/// #     nums.iter().sum::<f64>() / nums.len() as f64
/// # }
/// ```
fn average(nums: &[f64]) -> f64 {
    nums.iter().sum::<f64>() / nums.len() as f64
}
```

### Common Doc Sections

The Rust community uses standard section headers in doc comments:

```rust
/// Brief one-line description.
///
/// Longer description with more detail about what this
/// function does, any important notes, etc.
///
/// # Arguments
///
/// * `param1` - Description of first parameter
/// * `param2` - Description of second parameter
///
/// # Returns
///
/// Description of the return value.
///
/// # Errors
///
/// Describes when this function returns an `Err`.
/// (For functions that return `Result`)
///
/// # Panics
///
/// Describes conditions that cause this function to panic.
///
/// # Safety
///
/// Describes the invariants callers must uphold.
/// (For `unsafe` functions)
///
/// # Examples
///
/// ```
/// // Working example code here
/// ```
fn well_documented_function(param1: i32, param2: &str) -> Result<String, String> {
    Ok(format!("{}: {}", param2, param1))
}
```

| Section | When to Include |
|---------|----------------|
| `# Arguments` | When parameters aren't self-explanatory |
| `# Returns` | When the return value needs explanation |
| `# Errors` | When returning `Result` |
| `# Panics` | When the function can panic |
| `# Safety` | When the function is `unsafe` |
| `# Examples` | **Always** — examples are the most useful part! |

---

## Generating Documentation with `cargo doc`

Rust can generate beautiful HTML documentation from your doc comments:

```bash
# Generate docs
cargo doc

# Generate docs and open in browser
cargo doc --open

# Include documentation for dependencies too
cargo doc --document-private-items
```

The generated docs look like the standard library docs at [doc.rust-lang.org](https://doc.rust-lang.org/std/). Every crate published to crates.io automatically gets documentation at [docs.rs](https://docs.rs).

---

## Rust Naming Conventions

Rust has strong, consistent naming conventions enforced by the compiler (warnings) and `clippy`:

### The Complete Naming Table

| Item | Convention | Example |
|------|-----------|---------|
| Variables | `snake_case` | `let user_name = "Alice";` |
| Functions | `snake_case` | `fn calculate_area()` |
| Methods | `snake_case` | `rect.get_area()` |
| Macros | `snake_case!` | `println!()`, `vec![]` |
| Modules | `snake_case` | `mod file_utils;` |
| Crates | `snake_case` | `my_cool_crate` |
| Constants | `SCREAMING_SNAKE_CASE` | `const MAX_SIZE: usize = 100;` |
| Statics | `SCREAMING_SNAKE_CASE` | `static COUNTER: AtomicUsize = ...;` |
| Types (structs) | `PascalCase` | `struct UserProfile {}` |
| Types (enums) | `PascalCase` | `enum Color { Red, Green }` |
| Traits | `PascalCase` | `trait Drawable {}` |
| Type aliases | `PascalCase` | `type Meters = f64;` |
| Enum variants | `PascalCase` | `Color::DarkRed` |
| Generic types | Single uppercase letter | `fn foo<T>(x: T)` |
| Lifetimes | Short lowercase | `'a`, `'input`, `'static` |

### Why Naming Conventions Matter

```rust
// ❌ Inconsistent — hard to read, confusing
fn CalculateArea(Width: f64, Height: f64) -> f64 { Width * Height }
const max_retries: i32 = 3;
struct user_profile { UserName: String }

// ✅ Consistent — instantly readable
fn calculate_area(width: f64, height: f64) -> f64 { width * height }
const MAX_RETRIES: i32 = 3;
struct UserProfile { username: String }
```

The compiler warns you if you break conventions:

```
warning: variable `X` should have a snake case name
  --> src/main.rs:2:9
   |
2  |     let X = 5;
   |         ^ help: convert the identifier to snake case: `x`
```

### Common Naming Patterns in Rust

```rust
// Conversion methods: from_*, to_*, as_*, into_*
fn from_string(s: String) -> Self { ... }    // Creates from another type
fn to_string(&self) -> String { ... }        // Converts (may clone)
fn as_str(&self) -> &str { ... }             // Borrows as another type (cheap)
fn into_inner(self) -> T { ... }             // Consumes self, returns inner

// Boolean getters: is_*, has_*
fn is_empty(&self) -> bool { ... }
fn has_children(&self) -> bool { ... }

// Constructors: new, with_*
fn new() -> Self { ... }
fn with_capacity(cap: usize) -> Self { ... }

// Iterators: iter, iter_mut, into_iter
fn iter(&self) -> Iter<T> { ... }
fn iter_mut(&mut self) -> IterMut<T> { ... }
fn into_iter(self) -> IntoIter<T> { ... }
```

---

## Code Formatting with `rustfmt`

Rust has an official code formatter — `rustfmt`. It eliminates all style debates by enforcing one canonical style.

### Running rustfmt

```bash
# Format a single file
rustfmt src/main.rs

# Format the entire project (preferred)
cargo fmt

# Check formatting without changing files (useful in CI)
cargo fmt -- --check
```

### What rustfmt Does

```rust
// Before rustfmt (messy):
fn   main( ){let x=5;let y=10;if x>3{println!("{}",x)}    else{
  println!("nope")}let z = x+y;println!("z = {}",z);}

// After rustfmt (clean):
fn main() {
    let x = 5;
    let y = 10;
    if x > 3 {
        println!("{}", x)
    } else {
        println!("nope")
    }
    let z = x + y;
    println!("z = {}", z);
}
```

### Configuring rustfmt

Create a `rustfmt.toml` or `.rustfmt.toml` in your project root:

```toml
# rustfmt.toml
max_width = 100        # Default: 100 characters
tab_spaces = 4         # Default: 4 spaces
edition = "2021"       # Match your Rust edition
use_small_heuristics = "Default"
```

Most projects use the defaults. **The Rust standard is: don't customize, just use the defaults.** This ensures every Rust project looks the same.

> **Real-World Practice:** Set up your editor to run `rustfmt` on save. In VS Code with rust-analyzer, this happens automatically.

---

## Indentation and Braces

Rust uses **4 spaces** for indentation (not tabs, not 2 spaces). Braces go on the **same line** as the statement:

```rust
// ✅ Correct Rust style
fn calculate(x: i32) -> i32 {
    if x > 0 {
        x * 2
    } else {
        x * -1
    }
}

// ❌ Wrong: brace on next line (this is C# / Java style)
fn calculate(x: i32) -> i32
{
    if x > 0
    {
        x * 2
    }
    else
    {
        x * -1
    }
}
```

### Consistent Indentation

```rust
fn main() {
    let numbers = [1, 2, 3, 4, 5];     // 4 spaces
    
    for num in &numbers {                // 4 spaces
        if *num > 2 {                    // 8 spaces
            println!("{}", num);         // 12 spaces
        }                                // 8 spaces
    }                                    // 4 spaces
}
```

---

## Line Length

The standard maximum line length is **100 characters**. Long lines should be broken up:

```rust
// ❌ Too long
let result = some_function(first_argument, second_argument, third_argument, fourth_argument, fifth_argument);

// ✅ Broken across lines
let result = some_function(
    first_argument,
    second_argument,
    third_argument,
    fourth_argument,
    fifth_argument,
);

// ✅ Chained methods — each on its own line
let processed = data
    .iter()
    .filter(|x| x.is_valid())
    .map(|x| x.transform())
    .collect::<Vec<_>>();
```

### Trailing Commas

Rust style uses **trailing commas** in multi-line constructs:

```rust
// ✅ Trailing comma (makes diffs cleaner)
let colors = [
    "red",
    "green",
    "blue",   // ← trailing comma
];

fn process(
    input: &str,
    config: &Config,
    verbose: bool,   // ← trailing comma
) -> Result<(), Error> {
    // ...
    Ok(())
}
```

Why trailing commas? They make version control diffs cleaner:

```diff
  let colors = [
      "red",
      "green",
      "blue",
+     "yellow",
  ];
```

Without a trailing comma, you'd have to modify the "blue" line too:

```diff
  let colors = [
      "red",
      "green",
-     "blue"
+     "blue",
+     "yellow"
  ];
```

---

## Idiomatic Rust Style

### Use `if let` Instead of Single-Arm `match`

```rust
let value: Option<i32> = Some(42);

// ❌ Verbose
match value {
    Some(v) => println!("{}", v),
    None => {},
}

// ✅ Concise
if let Some(v) = value {
    println!("{}", v);
}
```

### Prefer Pattern Matching Over Indexing

```rust
let pair = (10, 20);

// ❌ Less readable
let x = pair.0;
let y = pair.1;

// ✅ More readable — destructuring
let (x, y) = pair;
```

### Use Iterators Instead of Manual Loops

```rust
let numbers = vec![1, 2, 3, 4, 5];

// ❌ C-style loop (works, but not idiomatic)
let mut sum = 0;
for i in 0..numbers.len() {
    sum += numbers[i];
}

// ✅ Idiomatic Rust
let sum: i32 = numbers.iter().sum();
```

### Return Without `return` When Possible

```rust
// ❌ Unnecessary explicit return
fn double(x: i32) -> i32 {
    return x * 2;
}

// ✅ Idiomatic: last expression is the return value
fn double(x: i32) -> i32 {
    x * 2
}
```

### Use `todo!()` as a Placeholder

```rust
fn parse_config(path: &str) -> Config {
    todo!("Implement config parsing")
    // Compiles but panics at runtime — useful during development
}
```

### Handle All Cases Explicitly

```rust
// ❌ Ignoring the Result (compiler warns)
let _ = std::fs::read_to_string("file.txt");

// ✅ Handle the error
match std::fs::read_to_string("file.txt") {
    Ok(contents) => println!("{}", contents),
    Err(e) => eprintln!("Error: {}", e),
}

// ✅ Or use expect() for quick prototyping
let contents = std::fs::read_to_string("file.txt")
    .expect("Failed to read file");
```

---

## When to Comment and When Not To

### Good Comments Explain WHY

```rust
// ✅ Good: explains WHY
// We retry 3 times because the upstream API occasionally drops connections
const MAX_RETRIES: u32 = 3;

// ✅ Good: explains a non-obvious decision
// Using i64 instead of i32 because user IDs can exceed 2 billion
let user_id: i64 = get_user_id();

// ✅ Good: documents a workaround
// HACK: The library doesn't handle empty strings, so we add a space
let input = if raw_input.is_empty() { " " } else { &raw_input };
```

### Bad Comments Restate the Code

```rust
// ❌ Bad: restates the obvious
let count = 0; // Initialize count to 0

// ❌ Bad: the code is self-explanatory
// Loop through all users
for user in &users {
    // Print the user's name
    println!("{}", user.name);
}

// ❌ Bad: outdated comment (worse than no comment!)
// Returns the user's age
fn get_user_name() -> String { ... }  // ← Comment says age, code says name!
```

### The Golden Rule

```
Good code + good names = rarely needs comments
Good code + good names + comments explaining WHY = excellent code
Bad code + lots of comments = still bad code (refactor instead!)
```

### TODO, FIXME, HACK Comments

Standard markers for work-in-progress:

```rust
// TODO: Implement caching for frequently accessed items
fn get_item(id: u64) -> Item {
    fetch_from_database(id) // Currently hits DB every time
}

// FIXME: This panics when the input contains Unicode
fn count_chars(s: &str) -> usize {
    s.len() // Should use s.chars().count()
}

// HACK: Temporary workaround for issue #123
fn normalize(value: f64) -> f64 {
    if value.is_nan() { 0.0 } else { value }
}

// NOTE: This function is called from multiple threads
fn shared_counter() -> usize { ... }

// SAFETY: The pointer is guaranteed to be valid by the caller
unsafe fn deref_raw(ptr: *const i32) -> i32 { *ptr }
```

---

## Real-World Code Style Examples

### Example: A Well-Styled Function

```rust
/// Finds the two numbers in the slice that sum to the target.
///
/// Uses a brute-force approach with O(n²) time complexity.
/// For larger inputs, consider using a HashMap-based approach.
///
/// # Arguments
///
/// * `numbers` - A slice of integers to search
/// * `target` - The desired sum
///
/// # Returns
///
/// `Some((index1, index2))` if a pair is found, `None` otherwise.
///
/// # Examples
///
/// ```
/// let nums = [2, 7, 11, 15];
/// assert_eq!(two_sum(&nums, 9), Some((0, 1)));
/// assert_eq!(two_sum(&nums, 100), None);
/// ```
fn two_sum(numbers: &[i32], target: i32) -> Option<(usize, usize)> {
    for i in 0..numbers.len() {
        for j in (i + 1)..numbers.len() {
            if numbers[i] + numbers[j] == target {
                return Some((i, j));
            }
        }
    }
    None
}
```

### Example: A Well-Structured Module

```rust
//! Utilities for working with temperature conversions.
//!
//! Supports Celsius, Fahrenheit, and Kelvin scales.

/// A temperature value with its scale.
///
/// # Examples
///
/// ```
/// let boiling = Temperature::celsius(100.0);
/// let freezing = Temperature::fahrenheit(32.0);
/// ```
struct Temperature {
    /// The temperature value in Celsius (internal representation).
    celsius: f64,
}

impl Temperature {
    /// Creates a new temperature from a Celsius value.
    fn celsius(value: f64) -> Self {
        Temperature { celsius: value }
    }

    /// Creates a new temperature from a Fahrenheit value.
    fn fahrenheit(value: f64) -> Self {
        Temperature {
            celsius: (value - 32.0) * 5.0 / 9.0,
        }
    }

    /// Converts this temperature to Fahrenheit.
    fn to_fahrenheit(&self) -> f64 {
        self.celsius * 9.0 / 5.0 + 32.0
    }

    /// Returns `true` if this temperature is below freezing (0°C).
    fn is_freezing(&self) -> bool {
        self.celsius < 0.0
    }
}
```

---

## Clippy — The Rust Linter

Clippy catches common mistakes and suggests more idiomatic code:

```bash
# Run Clippy
cargo clippy

# Run Clippy and fail on warnings (useful for CI)
cargo clippy -- -D warnings
```

### Examples of Clippy Suggestions

```rust
// Clippy: "this can be simplified"
// ❌ Before
if x == true { ... }
// ✅ After
if x { ... }

// Clippy: "use `is_empty()` instead"
// ❌ Before
if vec.len() == 0 { ... }
// ✅ After
if vec.is_empty() { ... }

// Clippy: "manual implementation of `map`"
// ❌ Before
let result = match value {
    Some(v) => Some(v * 2),
    None => None,
};
// ✅ After
let result = value.map(|v| v * 2);

// Clippy: "this loop could be written as a `for` loop"
// ❌ Before
let mut i = 0;
while i < arr.len() {
    process(arr[i]);
    i += 1;
}
// ✅ After
for item in &arr {
    process(*item);
}
```

> **Real-World Practice:** Always run `cargo clippy` before committing. Most teams enforce zero Clippy warnings in CI.

---

## Common Mistakes

### Mistake 1: Commenting Out Code Permanently

```rust
// ❌ Don't leave dead code in comments
fn process() {
    // let old_value = calculate_v1();
    // let result = old_value * 2;
    // println!("{}", result);
    
    let new_value = calculate_v2();
    println!("{}", new_value);
}

// ✅ Remove dead code — version control tracks history
fn process() {
    let new_value = calculate_v2();
    println!("{}", new_value);
}
```

### Mistake 2: Outdated Comments

```rust
// ❌ Comment says one thing, code does another
/// Returns the sum of two numbers
fn multiply(a: i32, b: i32) -> i32 {   // ← LIE!
    a * b
}

// ✅ Keep comments in sync with code
/// Returns the product of two numbers
fn multiply(a: i32, b: i32) -> i32 {
    a * b
}
```

### Mistake 3: Inconsistent Naming

```rust
// ❌ Mixing styles
let userName = "Alice";     // camelCase (JavaScript style)
let user_age = 30;          // snake_case (Rust style)
let USERCOUNT = 100;        // ALLCAPS (constant style for a variable)

// ✅ Consistent snake_case for variables
let user_name = "Alice";
let user_age = 30;
let user_count = 100;
```

### Mistake 4: Not Running `cargo fmt`

```rust
// ❌ Before formatting (inconsistent spacing)
fn main(){
let x=5;
  let y = 10;
    println!("{}",x+y);
}

// ✅ After `cargo fmt`
fn main() {
    let x = 5;
    let y = 10;
    println!("{}", x + y);
}
```

Just run `cargo fmt`. Every time. Before every commit.

---

## Exercises

### Exercise 1: Add Doc Comments

Take this function and add proper doc comments with `///`, including a description, `# Arguments`, `# Returns`, and `# Examples` section:

```rust
fn is_palindrome(s: &str) -> bool {
    let cleaned: String = s.chars()
        .filter(|c| c.is_alphanumeric())
        .map(|c| c.to_lowercase().next().unwrap())
        .collect();
    cleaned == cleaned.chars().rev().collect::<String>()
}
```

### Exercise 2: Fix the Style

Rewrite this code following Rust naming conventions and style guidelines:

```rust
fn CalculateAverage(Numbers: &[f64]) -> f64 {
let mut Total = 0.0;
let mut Count = 0;
for n in Numbers { Total += n; Count += 1; }
return Total / Count as f64;
}
```

### Exercise 3: Comment Review

Identify which of these comments are good and which should be removed or rewritten:

```rust
let x = 5; // set x to 5
// Using binary search because the list is sorted
let idx = binary_search(&list, target);
let mut count = 0; // initialize count
// HACK: API returns inconsistent date formats, normalize here
let date = normalize_date(raw_date);
```

### Exercise 4: Write Module Documentation

Create a file with `//!` inner doc comments that documents a hypothetical "string_utils" module. Include a description, feature list, and quick start example.

### Exercise 5: Run the Tools

If you have a Rust project:
1. Run `cargo fmt` and see what changes
2. Run `cargo clippy` and fix all warnings
3. Run `cargo doc --open` and browse the generated docs

---

## Summary

### Comment Types

| Syntax | Name | Purpose |
|--------|------|---------|
| `//` | Line comment | General comments |
| `/* */` | Block comment | Multi-line comments (nestable) |
| `///` | Outer doc comment | Document the next item |
| `//!` | Inner doc comment | Document the enclosing item |

### Naming Conventions

| Item | Style | Example |
|------|-------|---------|
| Variables, functions | `snake_case` | `let user_name`, `fn get_area()` |
| Types, traits | `PascalCase` | `struct MyStruct`, `trait Drawable` |
| Constants, statics | `SCREAMING_SNAKE_CASE` | `const MAX_SIZE: usize = 100` |
| Generics | Single uppercase | `<T>`, `<K, V>` |
| Lifetimes | Short, lowercase with `'` | `'a`, `'static` |

### Essential Tools

| Tool | Command | Purpose |
|------|---------|---------|
| `rustfmt` | `cargo fmt` | Format code to standard style |
| `clippy` | `cargo clippy` | Lint for common mistakes |
| `cargo doc` | `cargo doc --open` | Generate HTML docs |

### Key Takeaways

1. **Use `//` for most comments** — `/* */` is rare in Rust
2. **Use `///` doc comments** on all public items — they become HTML docs
3. **Comment WHY, not WHAT** — good names make WHAT obvious
4. **Run `cargo fmt` on every save** — no debate, one standard style
5. **Run `cargo clippy` before committing** — catches common issues
6. **Follow naming conventions** — the compiler warns you if you don't
7. **Delete dead code** — version control remembers it for you
8. **Keep comments up to date** — an outdated comment is worse than none

---

## Stage 2 Complete! 🎉

You've finished all 8 tutorials in **Rust Fundamentals**. You now know:

- ✅ Variables, mutability, and shadowing
- ✅ Constants and statics
- ✅ All scalar types (integers, floats, booleans, characters)
- ✅ Compound types (tuples and arrays)
- ✅ Functions (parameters, return values, expressions)
- ✅ Control flow — conditionals (if/else, match, if let)
- ✅ Control flow — loops (loop, while, for, labels)
- ✅ Comments and code style

**Next Stage:** [Stage 3: Ownership — Rust's Superpower →](../03-ownership-the-superpower/README.md)

---

<p align="center">
  <i>Tutorial 8 of 8 — Stage 2: Rust Fundamentals</i>
</p>

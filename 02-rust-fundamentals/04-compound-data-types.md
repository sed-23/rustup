# Compound Data Types 📦📦

> **Compound types group multiple values into one type. Rust has two primitive compound types: tuples and arrays.**

---

## Table of Contents

- [What Are Compound Types?](#what-are-compound-types)
- [Tuples](#tuples)
  - [Creating Tuples](#creating-tuples)
  - [Accessing Tuple Elements — Dot Notation](#accessing-tuple-elements--dot-notation)
  - [Destructuring Tuples](#destructuring-tuples)
  - [Tuples in Memory](#tuples-in-memory)
  - [The Unit Type — Empty Tuple](#the-unit-type--empty-tuple)
  - [Tuples as Function Returns](#tuples-as-function-returns)
  - [Tuple Limitations](#tuple-limitations)
- [Arrays](#arrays)
  - [Creating Arrays](#creating-arrays)
  - [Accessing Array Elements — Index Notation](#accessing-array-elements--index-notation)
  - [Arrays in Memory](#arrays-in-memory)
  - [Array Length and Iteration](#array-length-and-iteration)
  - [Mutable Arrays](#mutable-arrays)
  - [Out-of-Bounds Access — Runtime Panic](#out-of-bounds-access--runtime-panic)
  - [Array Slices — Borrowing Part of an Array](#array-slices--borrowing-part-of-an-array)
  - [Multi-Dimensional Arrays](#multi-dimensional-arrays)
  - [Arrays vs Vectors](#arrays-vs-vectors)
- [Practical Patterns](#practical-patterns)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Are Compound Types?

So far we've seen **scalar types** — they hold a single value (`i32`, `f64`, `bool`, `char`). **Compound types** group multiple values together:

```
Types in Rust
├── Scalar (single value)
│   ├── Integer
│   ├── Float
│   ├── Boolean
│   └── Character
│
└── Compound (multiple values)
    ├── Tuple   → fixed size, mixed types
    └── Array   → fixed size, same type
```

| Feature | Tuple | Array |
|---------|-------|-------|
| Mixed types? | ✅ Yes | ❌ No (all same type) |
| Fixed size? | ✅ Yes | ✅ Yes |
| Access by | Index (dot notation): `.0`, `.1` | Index (bracket notation): `[0]`, `[1]` |
| Growable? | ❌ No | ❌ No (use `Vec` for that) |

### Where Do Compound Types Come From?

**Tuples** were invented in mathematics long before computers. The word comes from Latin: single, double, triple, quadruple, quintuple... N-tuple. In set theory, a tuple is an **ordered sequence** of elements. The key insight is "ordered" — `(3, 5)` is different from `(5, 3)`, unlike a set where `{3, 5}` = `{5, 3}`.

Programming languages borrowed tuples from mathematics:
- **LISP (1958):** Used linked list pairs (cons cells) as a primitive form of tuples
- **ML (1973):** Introduced true tuples as a built-in type — Rust's tuples are directly descended from ML's
- **Python (1991):** Made tuples popular with simple syntax: `(1, "hello", True)`
- **Haskell (1990):** Uses tuples extensively in its type system
- **Rust (2015):** Follows the ML/Haskell tradition with typed tuples

**Arrays** evolved from the mathematical concept of a **sequence** or **vector**. They were one of the first data structures in programming:
- **Fortran (1957):** Had arrays as its ONLY data structure — "Fortran" literally means "Formula Translation," and arrays were essential for scientific computing
- **C (1972):** Arrays are just pointer arithmetic — `arr[i]` is literally `*(arr + i)`. No bounds checking!
- **Java (1995):** Arrays are objects on the heap with built-in bounds checking
- **Rust (2015):** Arrays are stack-allocated with compile-time known size, and bounds checking at runtime

> **Design Philosophy:** Rust's compound types follow a principle of **zero-cost abstraction** — tuples and arrays compile to the exact same memory layout as raw data in C, but with type safety and bounds checking added by the compiler. You get safety without paying a performance price.

---

## Tuples

A **tuple** is an ordered, fixed-size collection of values that can have **different types**.

### Creating Tuples

```rust
fn main() {
    // A tuple with three different types
    let person: (String, i32, bool) = (String::from("Alice"), 30, true);
    
    // Type annotation is optional — Rust can infer it
    let point = (10, 20);               // (i32, i32)
    let mixed = (500, 6.4, true, 'A');  // (i32, f64, bool, char)
    let single = (42,);                 // Single-element tuple — NOTE the comma!
    
    println!("Person: {:?}", person);
    println!("Point: {:?}", point);
    println!("Mixed: {:?}", mixed);
}
```

Output:
```
Person: ("Alice", 30, true)
Point: (10, 20)
Mixed: (500, 6.4, true, 'A')
```

### The Syntax

```
let tuple_name: (Type1, Type2, Type3) = (value1, value2, value3);
                 │                        │
                 └── Type annotation       └── Values in parentheses
                    (optional)                 separated by commas
```

### Single-Element Tuple — The Trailing Comma

```rust
fn main() {
    let not_a_tuple = (42);   // This is just an i32! Parentheses are just grouping.
    let is_a_tuple = (42,);   // THIS is a tuple with one element
    
    println!("Type check:");
    println!("(42) is i32: {}", not_a_tuple);
    // not_a_tuple.0  // ❌ ERROR: i32 has no field 0
    println!("(42,).0 = {}", is_a_tuple.0);  // ✅ Works: 42
}
```

### Accessing Tuple Elements — Dot Notation

Use a period followed by the index (starting from 0):

```rust
fn main() {
    let person = ("Alice", 30, true);
    
    let name = person.0;       // "Alice"  (first element, index 0)
    let age = person.1;        // 30       (second element, index 1)
    let is_active = person.2;  // true     (third element, index 2)
    
    println!("Name: {}", name);
    println!("Age: {}", age);
    println!("Active: {}", is_active);
}
```

```
Tuple: ("Alice", 30, true)
Index:    .0      .1   .2
```

### Why Dot Notation Instead of Bracket `[]`?

Bracket notation (`person[0]`) is for arrays and vectors where the index can be a runtime variable. Tuple indices MUST be literal numbers known at compile time:

```rust
fn main() {
    let tup = (1, 2, 3);
    
    let first = tup.0;  // ✅ Literal index — works
    
    // let i = 0;
    // let val = tup.i;  // ❌ ERROR: no field `i` on type `(i32, i32, i32)`
    
    // Why? Because each position in a tuple can have a DIFFERENT type.
    // The compiler must know at compile time which type to return.
}
```

### Destructuring Tuples

**Destructuring** means breaking a tuple into its individual parts:

```rust
fn main() {
    let person = ("Alice", 30, true);
    
    // Destructure into three variables
    let (name, age, is_active) = person;
    
    println!("Name: {}", name);       // Alice
    println!("Age: {}", age);         // 30
    println!("Active: {}", is_active); // true
}
```

### How Destructuring Works

```
let (name,  age,  is_active) = ("Alice", 30, true);
     │       │      │            │        │    │
     │       │      └────────────│────────│────┘
     │       └───────────────────│────────┘
     └───────────────────────────┘
     
name = "Alice"
age = 30
is_active = true
```

### Partial Destructuring with `_`

If you don't need all elements, use `_` to ignore them:

```rust
fn main() {
    let rgb = (255, 128, 0);
    
    let (red, _, _) = rgb;        // Only keep the first
    println!("Red: {}", red);
    
    let (_, green, _) = rgb;      // Only keep the second
    println!("Green: {}", green);
    
    let (_, _, blue) = rgb;       // Only keep the third
    println!("Blue: {}", blue);
    
    // Use .. to ignore the rest (in patterns — more in Stage 6)
    let (first, ..) = rgb;
    println!("First: {}", first);
}
```

### Tuples in Memory

Tuples are stored **contiguously** on the stack (one element right after another), with possible **padding** for alignment:

```rust
fn main() {
    let t: (u8, i32, bool) = (65, 1000, true);
    
    println!("Size of (u8, i32, bool): {} bytes", 
        std::mem::size_of::<(u8, i32, bool)>());
    // Output: 8 bytes
}
```

Wait — `u8` (1 byte) + `i32` (4 bytes) + `bool` (1 byte) = 6 bytes. Why does Rust say 8?

```
Memory Layout (with alignment padding):

Address:  0x00  0x01  0x02  0x03  0x04  0x05  0x06  0x07
          ┌─────┬─────┬─────┬─────┬─────────────────────┐
          │ u8  │ pad │ pad │ pad │  i32 (4 bytes)       │
          │ 65  │     │     │     │  1000                │
          └─────┴─────┴─────┴─────┴──────┬──────┬───────┘
                                          │ bool │ pad  
                                          │ true │
                                          └──────┘

The i32 must be aligned to a 4-byte boundary, so 3 padding bytes
are inserted after the u8.
```

> **Note:** The Rust compiler may **reorder** tuple fields for better alignment, so the actual memory layout might differ from what you'd expect. This is an implementation detail you normally don't need to worry about.

### Deep Dive: Why Does Alignment Matter?

**Memory alignment** is one of those low-level details that most languages hide from you, but understanding it explains why data structures sometimes use more memory than expected.

Modern CPUs don't read memory one byte at a time — they read in **chunks** (called "words"), typically 4 or 8 bytes at a time. For efficiency, data should be aligned to its size:

- An `i32` (4 bytes) should start at an address divisible by 4
- An `i64` (8 bytes) should start at an address divisible by 8
- A `bool` (1 byte) can start anywhere

```
Why alignment matters:

Address: 0  1  2  3  4  5  6  7  8  9  10 11
         ├──┴──┴──┴──┤  ├──┴──┴──┴──┤
         CPU Read #1     CPU Read #2

         ALIGNED i32 at address 0:
         [0][1][2][3]  ← One read gets all 4 bytes ✅

         MISALIGNED i32 at address 1:
            [1][2][3][4]  ← Spans TWO reads! The CPU must:
                            1. Read bytes 0-3 (gets bytes 1-3)
                            2. Read bytes 4-7 (gets byte 4)
                            3. Combine them
                            This is 2-3x SLOWER on most CPUs!
```

Some architectures (like ARM) will actually **crash** on misaligned access. x86 handles it but with a performance penalty. Rust's compiler automatically inserts **padding bytes** to ensure all fields are properly aligned.

This is why `(u8, i32, bool)` takes 8 bytes instead of 6 — the 3 padding bytes after `u8` ensure the `i32` starts at a 4-byte-aligned address. It's wasted space, but it makes access much faster.

> **Comparison:** In C, you can control struct packing with `__attribute__((packed))` to remove padding (at the cost of speed). Rust has `#[repr(packed)]` for the same purpose, but it's rarely used outside of FFI (foreign function interface) code.

### The Unit Type — Empty Tuple

The **unit type** `()` is a tuple with zero elements. It represents "nothing" or "no value":

```rust
fn main() {
    let unit: () = ();
    
    println!("Size of (): {} bytes", std::mem::size_of::<()>());
    // Output: 0 bytes — it takes NO memory!
}
```

Where is `()` used?

```rust
// 1. Functions that don't return a value implicitly return ()
fn greet() {
    println!("Hello!");
    // implicitly returns ()
}

// This is equivalent to:
fn greet_explicit() -> () {
    println!("Hello!");
}

// 2. Expressions that produce no useful value
fn main() {
    let x = {
        println!("inside block");
        // This block doesn't have a final expression, so it returns ()
    };
    // x is ()
    
    // 3. The semicolon turns an expression into a statement (returns ())
    let y = if true { 5 } else { 10 };    // y = 5
    // but:
    let z: () = if true { 5; } else { 10; };  // z = () (semicolons discard values)
}
```

### Tuples as Function Returns

Tuples are great for returning multiple values from a function:

```rust
fn divide(dividend: i32, divisor: i32) -> (i32, i32) {
    let quotient = dividend / divisor;
    let remainder = dividend % divisor;
    (quotient, remainder)
}

fn min_max(numbers: &[i32]) -> (i32, i32) {
    let mut min = numbers[0];
    let mut max = numbers[0];
    for &n in numbers {
        if n < min { min = n; }
        if n > max { max = n; }
    }
    (min, max)
}

fn main() {
    let (q, r) = divide(17, 5);
    println!("17 / 5 = {} remainder {}", q, r);  // 3 remainder 2
    
    let numbers = [3, 1, 4, 1, 5, 9, 2, 6];
    let (min, max) = min_max(&numbers);
    println!("Min: {}, Max: {}", min, max);  // Min: 1, Max: 9
}
```

### Tuple Limitations

1. **Maximum practical size**: Tuples can technically be any size, but standard library traits (like `Debug`, `Clone`, etc.) are only implemented for tuples up to 12 elements
2. **No iteration**: You can't loop over tuple elements (because they can have different types)
3. **No named fields**: `.0`, `.1`, `.2` aren't descriptive — use a `struct` (Stage 4) for clarity

```rust
// ❌ Hard to read
let user = ("Alice", 30, "alice@example.com", true, 1000);
let email = user.2;  // What's at index 2? 🤷

// ✅ Better — use a struct (Stage 4)
// struct User { name: String, age: i32, email: String, active: bool, balance: i32 }
// let user = User { name: "Alice".into(), age: 30, email: "alice@example.com".into(), ... };
// let email = user.email;  // Much clearer!
```

---

## Arrays

An **array** is a fixed-size collection where **all elements have the same type**.

### Creating Arrays

```rust
fn main() {
    // Type: [i32; 5] — an array of 5 i32 values
    let numbers: [i32; 5] = [1, 2, 3, 4, 5];
    
    // Type inference works
    let fruits = ["apple", "banana", "cherry"];  // [&str; 3]
    
    // Initialize all elements to the same value
    let zeros = [0; 5];        // [0, 0, 0, 0, 0]
    let ones = [1.0; 3];       // [1.0, 1.0, 1.0]
    let dashes = ['-'; 40];    // 40 dashes
    
    println!("numbers: {:?}", numbers);
    println!("fruits: {:?}", fruits);
    println!("zeros: {:?}", zeros);
}
```

Output:
```
numbers: [1, 2, 3, 4, 5]
fruits: ["apple", "banana", "cherry"]
zeros: [0, 0, 0, 0, 0]
```

### The Syntax

```
let name: [Type; LENGTH] = [value1, value2, ...];
           │      │
           │      └── Number of elements (must be known at compile time)
           └──────── Type of each element

let name = [initial_value; LENGTH];
            │               │
            └── Every element gets this value
                            └── How many elements
```

### Important: Array Length is Part of the Type!

```rust
fn main() {
    let a: [i32; 3] = [1, 2, 3];
    let b: [i32; 5] = [1, 2, 3, 4, 5];
    
    // a and b are DIFFERENT types!
    // [i32; 3] ≠ [i32; 5]
    
    // This means:
    // let c: [i32; 3] = b;  // ❌ ERROR: expected [i32; 3], found [i32; 5]
}
```

This is unlike most other languages where array length is not part of the type:

```
C:          int arr[5];      // Length not in type system
Java:       int[] arr;       // Length not in type
Python:     arr = [1,2,3]    // No type system
JavaScript: let arr = [1,2,3] // No type system
Rust:       let arr: [i32; 5] // Length IS part of the type!
```

### Accessing Array Elements — Index Notation

Use square brackets `[]` with a `usize` index (starting from 0):

```rust
fn main() {
    let days = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"];
    
    let first = days[0];    // "Mon"
    let third = days[2];    // "Wed"
    let last = days[6];     // "Sun"
    
    println!("First day: {}", first);
    println!("Third day: {}", third);
    println!("Last day: {}", last);
    
    // The index MUST be usize
    let i: usize = 3;
    println!("Day at index {}: {}", i, days[i]);  // "Thu"
    
    // Other integer types won't work:
    // let j: i32 = 3;
    // println!("{}", days[j]);  // ❌ ERROR: expected usize, found i32
}
```

```
Array: ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
Index:    0      1      2      3      4      5      6
```

### Arrays in Memory

Arrays are stored **contiguously** on the stack — all elements are right next to each other in memory:

```rust
fn main() {
    let arr: [i32; 5] = [10, 20, 30, 40, 50];
    
    println!("Size: {} bytes", std::mem::size_of_val(&arr));
    // Output: 20 bytes (5 × 4 bytes per i32)
}
```

```
Stack Memory:

Address:  0x100  0x104  0x108  0x10C  0x110
          ┌──────┬──────┬──────┬──────┬──────┐
          │  10  │  20  │  30  │  40  │  50  │
          │ [0]  │ [1]  │ [2]  │ [3]  │ [4]  │
          └──────┴──────┴──────┴──────┴──────┘
          ← 4B → ← 4B → ← 4B → ← 4B → ← 4B →
          └─────────────── 20 bytes ────────────┘
```

To access element at index `i`:
$$\text{address} = \text{base\_address} + i \times \text{element\_size}$$

Example: `arr[3]` → `0x100 + 3 × 4 = 0x10C` → value `40`

This is why array access is $O(1)$ (constant time) — the CPU can calculate any element's address instantly, regardless of array size.

### Array Length and Iteration

```rust
fn main() {
    let numbers = [10, 20, 30, 40, 50];
    
    // Get the length
    println!("Length: {}", numbers.len());  // 5
    
    // Check if empty
    println!("Is empty: {}", numbers.is_empty());  // false
    
    // Iterate with for loop (most common)
    for num in numbers {
        print!("{} ", num);
    }
    println!();
    // Output: 10 20 30 40 50
    
    // Iterate with index
    for i in 0..numbers.len() {
        println!("Index {}: {}", i, numbers[i]);
    }
    
    // Iterate with both index and value
    for (i, num) in numbers.iter().enumerate() {
        println!("[{}] = {}", i, num);
    }
    
    // Iterate over references (doesn't consume the array)
    for num in &numbers {
        print!("{} ", num);
    }
    println!();
}
```

### Mutable Arrays

```rust
fn main() {
    let mut scores = [0; 5];  // [0, 0, 0, 0, 0]
    
    scores[0] = 95;
    scores[1] = 87;
    scores[2] = 92;
    scores[3] = 78;
    scores[4] = 88;
    
    println!("Scores: {:?}", scores);
    // [95, 87, 92, 78, 88]
    
    // Calculate average
    let mut sum = 0;
    for score in &scores {
        sum += score;
    }
    let average = sum as f64 / scores.len() as f64;
    println!("Average: {:.1}", average);
    // Average: 88.0
}
```

### Important: `mut` Makes ALL Elements Mutable

You can't make individual elements mutable — it's all or nothing:

```rust
fn main() {
    let mut arr = [1, 2, 3];
    arr[0] = 10;  // ✅ Works because arr is mut
    arr[1] = 20;  // ✅ All elements are mutable
    
    let arr2 = [1, 2, 3];
    // arr2[0] = 10;  // ❌ arr2 is immutable
}
```

### Out-of-Bounds Access — Runtime Panic

Rust **checks array bounds at runtime** and crashes (panics) if you go out of bounds:

```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];
    
    // ❌ This compiles but PANICS at runtime
    let index = 10;
    // let val = arr[index];
    // thread 'main' panicked at 'index out of bounds: 
    //   the len is 5 but the index is 10'
}
```

This is **much safer** than C, where out-of-bounds access silently reads garbage memory:

```
C:    arr[10] → reads random memory (buffer overflow vulnerability!)
Rust: arr[10] → program panics with clear error message (safe!)
```

### Safe Alternative: `.get()`

```rust
fn main() {
    let arr = [10, 20, 30, 40, 50];
    
    // .get() returns Option<&T> — None if out of bounds
    match arr.get(2) {
        Some(value) => println!("Index 2: {}", value),  // Index 2: 30
        None => println!("Index out of bounds!"),
    }
    
    match arr.get(10) {
        Some(value) => println!("Index 10: {}", value),
        None => println!("Index 10 out of bounds!"),     // This prints
    }
    
    // Or with if let:
    if let Some(val) = arr.get(3) {
        println!("Found: {}", val);  // Found: 40
    }
}
```

### Array Slices — Borrowing Part of an Array

A **slice** is a reference to a contiguous portion of an array:

```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];
    
    // Full array as a slice
    let all: &[i32] = &arr;
    
    // Partial slices using range syntax
    let first_three: &[i32] = &arr[0..3];   // [1, 2, 3]
    let middle: &[i32] = &arr[1..4];        // [2, 3, 4]
    let last_two: &[i32] = &arr[3..5];      // [4, 5]
    let last_two_alt: &[i32] = &arr[3..];   // [4, 5] (to the end)
    let first_three_alt: &[i32] = &arr[..3]; // [1, 2, 3] (from the start)
    
    println!("All: {:?}", all);
    println!("First three: {:?}", first_three);
    println!("Middle: {:?}", middle);
    println!("Last two: {:?}", last_two);
}
```

### Range Syntax

```
[start..end]   → From start (inclusive) to end (exclusive)
[start..]      → From start to the end
[..end]        → From the beginning to end (exclusive)
[..]           → Everything (full slice)
[start..=end]  → From start (inclusive) to end (INCLUSIVE)
```

### Slice Type: `&[T]`

A slice `&[i32]` is different from an array `[i32; 5]`:

```rust
// Array: fixed size, known at compile time
fn print_array(arr: [i32; 5]) {
    // Can ONLY accept arrays of exactly 5 elements
}

// Slice: any size, known at runtime
fn print_slice(slice: &[i32]) {
    // Can accept a reference to ANY size array (or part of one)
    for item in slice {
        print!("{} ", item);
    }
    println!();
}

fn main() {
    let small = [1, 2, 3];
    let large = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    print_slice(&small);   // ✅ Works with 3 elements
    print_slice(&large);   // ✅ Works with 10 elements
    print_slice(&large[2..5]);  // ✅ Works with a sub-slice
}
```

> **Real-World Practice:** Functions should accept `&[T]` (slices) instead of `[T; N]` (arrays) to be flexible. This is one of Rust's most common patterns.

### How Slices Work in Memory

A slice is a "fat pointer" — it contains two pieces of information:

```
&[i32] = (pointer, length)

        Slice:
        ┌────────────────┬────────┐
        │ ptr: 0x104     │ len: 3 │   ← 16 bytes on 64-bit (8 + 8)
        └───────┬────────┴────────┘
                │
                ▼
Array:  ┌──────┬──────┬──────┬──────┬──────┐
        │  10  │  20  │  30  │  40  │  50  │
        │ [0]  │ [1]  │ [2]  │ [3]  │ [4]  │
        └──────┴──────┴──────┴──────┴──────┘
        0x100  0x104  0x108  0x10C  0x110

&arr[1..4] → pointer to arr[1] (0x104), length = 3
```

### Deep Dive: Fat Pointers — Rust's Secret Weapon

A regular pointer (like in C) is just an address — a single number pointing to a location in memory. A **fat pointer** carries extra metadata alongside the address.

Rust uses fat pointers in two places:
1. **Slices** (`&[T]`): pointer + length
2. **Trait objects** (`&dyn Trait`): pointer + vtable pointer (covered in Stage 8)

```
Regular pointer (C-style):
┌──────────┐
│ address  │  ← 8 bytes on 64-bit
└──────────┘

Fat pointer (Rust slice):
┌──────────┬──────────┐
│ address  │ length   │  ← 16 bytes on 64-bit (8 + 8)
└──────────┴──────────┘
```

Why does Rust bundle the length with the pointer? Consider what happens in C:

```c
// C: the function has NO IDEA how long the array is
void process(int* arr) {
    // How many elements? 🤷 No way to know!
    // arr[100] might be valid, or might be a buffer overflow
}

// C workaround: pass the length separately
void process(int* arr, size_t len) {
    // Now we know, but nothing FORCES the caller to pass the right length
}
```

In Rust, the slice carries its length, so bounds checking is always possible:
```rust
// Rust: the slice KNOWS its length
fn process(arr: &[i32]) {
    // arr.len() always returns the correct length
    // arr[100] will panic if out of bounds — not silently corrupt memory
}
```

This is one of the key ways Rust prevents **buffer overflows** — the vulnerability behind countless security exploits (the Morris Worm, Heartbleed, WannaCry, and many more were caused by reading/writing beyond array bounds in C).

### Multi-Dimensional Arrays

```rust
fn main() {
    // 2D array: 3 rows × 4 columns
    let matrix: [[i32; 4]; 3] = [
        [1,  2,  3,  4],
        [5,  6,  7,  8],
        [9, 10, 11, 12],
    ];
    
    // Access: matrix[row][col]
    println!("Row 0, Col 2: {}", matrix[0][2]);  // 3
    println!("Row 2, Col 3: {}", matrix[2][3]);  // 12
    
    // Print the whole matrix
    for row in &matrix {
        for val in row {
            print!("{:3} ", val);
        }
        println!();
    }
}
```

Output:
```
Row 0, Col 2: 3
Row 2, Col 3: 12
  1   2   3   4
  5   6   7   8
  9  10  11  12
```

Memory layout (row-major — rows are stored contiguously):
```
Address:  0x00  0x04  0x08  0x0C  0x10  0x14  0x18  0x1C  0x20  0x24  0x28  0x2C
          ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
          │  1  │  2  │  3  │  4  │  5  │  6  │  7  │  8  │  9  │ 10  │ 11  │ 12  │
          └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
          ← ─── Row 0 ─── → ← ─── Row 1 ─── → ← ─── Row 2 ─── →
```

### Arrays vs Vectors

Arrays and Vectors (`Vec<T>`) are both sequences of same-type elements, but they serve different purposes:

| Feature | Array `[T; N]` | Vector `Vec<T>` |
|---------|----------------|-----------------|
| Size | Fixed (compile time) | Dynamic (can grow/shrink) |
| Stored on | Stack | Heap |
| Speed | Faster (stack access) | Slightly slower (heap + indirection) |
| Can add elements? | ❌ No | ✅ Yes (`push`, `insert`) |
| Can remove elements? | ❌ No | ✅ Yes (`pop`, `remove`) |
| Use when | Size is known and fixed | Size may change |

```rust
fn main() {
    // Array — good for fixed-size data
    let weekdays = ["Mon", "Tue", "Wed", "Thu", "Fri"];
    let rgb: [u8; 3] = [255, 128, 0];
    
    // Vector — good for dynamic data (covered in Stage 4!)
    let mut shopping_list = vec!["eggs", "milk", "bread"];
    shopping_list.push("butter");
    shopping_list.push("cheese");
    println!("List: {:?}", shopping_list);
}
```

> **Real-World Practice:** Use arrays for small, fixed-size collections (RGB colors, coordinates, days of the week). Use `Vec<T>` for everything else. Most Rust code uses vectors far more often than arrays.

---

## Practical Patterns

### Pattern 1: RGB Color as Tuple or Array

```rust
fn main() {
    // As a tuple (mixed types possible)
    let color: (u8, u8, u8) = (255, 128, 0);
    println!("R: {}, G: {}, B: {}", color.0, color.1, color.2);
    
    // As an array (same type, indexable)
    let color: [u8; 3] = [255, 128, 0];
    for channel in &color {
        print!("{} ", channel);
    }
    println!();
    
    // With alpha (RGBA)
    let color: (u8, u8, u8, f32) = (255, 128, 0, 0.8);
    // Tuple is better here because alpha is a different type
}
```

### Pattern 2: Swap with Destructuring

```rust
fn main() {
    let mut a = 10;
    let mut b = 20;
    
    println!("Before: a = {}, b = {}", a, b);
    
    // Swap using tuple destructuring
    (a, b) = (b, a);
    
    println!("After:  a = {}, b = {}", a, b);
    // After: a = 20, b = 10
}
```

### Pattern 3: Array of Constants

```rust
const DIRECTIONS: [(i32, i32); 4] = [
    (0, -1),  // Up
    (0, 1),   // Down
    (-1, 0),  // Left
    (1, 0),   // Right
];

fn main() {
    let (x, y) = (5, 5);
    
    println!("From ({}, {}), neighbors are:", x, y);
    for (dx, dy) in &DIRECTIONS {
        println!("  ({}, {})", x + dx, y + dy);
    }
}
```

### Pattern 4: Lookup Tables

```rust
fn main() {
    // Days in each month (index 0 = January)
    let days_in_month: [u8; 12] = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];
    
    let month = 2;  // March (0-indexed)
    println!("March has {} days", days_in_month[month]);
    
    // Total days in a year
    let total: u16 = days_in_month.iter().map(|&d| d as u16).sum();
    println!("Total days: {}", total);  // 365
}
```

---

## Common Mistakes

### Mistake 1: Array Size Mismatch

```rust
// let arr: [i32; 3] = [1, 2, 3, 4];  // ❌ Expected 3 elements, found 4
let arr: [i32; 4] = [1, 2, 3, 4];     // ✅
```

### Mistake 2: Indexing with Wrong Type

```rust
let arr = [1, 2, 3];
let i: i32 = 1;
// let val = arr[i];  // ❌ expected usize, found i32
let val = arr[i as usize];  // ✅
```

### Mistake 3: Forgetting the Comma in Single-Element Tuple

```rust
let not_tuple = (42);    // This is i32, not a tuple!
let is_tuple = (42,);    // ✅ This IS a tuple
```

### Mistake 4: Trying to Resize an Array

```rust
let mut arr = [1, 2, 3];
// arr.push(4);  // ❌ Arrays don't have push — use Vec<T> instead
```

### Mistake 5: Confusing Array Type Notation

```rust
let a: [i32; 5];    // ✅ Array of 5 i32s
// let b: [5; i32];  // ❌ Wrong order! Type comes first, then size
```

---

## Exercises

### Exercise 1: Tuple Basics

Create a tuple representing a book: (title, author, year, pages, is_fiction). Destructure it and print each field on its own line.

### Exercise 2: Array Statistics

Create an array of 10 integers. Calculate and print:
- The sum
- The average (as f64)
- The maximum value
- The minimum value

### Exercise 3: 2D Grid

Create a 3×3 array initialized to 0. Set the diagonal elements to 1 (creating an identity matrix). Print the result:

```
1 0 0
0 1 0
0 0 1
```

### Exercise 4: Tuple Return

Write a function `analyze_text` that takes a `&str` and returns a tuple `(usize, usize, usize)` with the (number of characters, number of words, number of lines — count by splitting on spaces and newlines).

### Exercise 5: Array Reversal

Create an array `[1, 2, 3, 4, 5]` and write code to print it reversed (`5, 4, 3, 2, 1`) without using `.reverse()` or `.rev()`.

### Exercise 6: Slice Practice

Given the array `[10, 20, 30, 40, 50, 60, 70, 80, 90, 100]`:
1. Create a slice of the first 3 elements
2. Create a slice of the last 3 elements
3. Create a slice of elements at indices 3 through 6
4. Print the sum of each slice

---

## Summary

### Tuples

| Feature | Detail |
|---------|--------|
| Syntax | `let t: (i32, f64, bool) = (1, 2.0, true);` |
| Access | `t.0`, `t.1`, `t.2` (dot notation, compile-time index) |
| Destructure | `let (a, b, c) = t;` |
| Mixed types | ✅ Yes |
| Size | Fixed at compile time |
| Max practical size | 12 elements (for std traits) |
| Unit type | `()` — empty tuple, zero size |

### Arrays

| Feature | Detail |
|---------|--------|
| Syntax | `let a: [i32; 5] = [1, 2, 3, 4, 5];` |
| Fill syntax | `let a = [0; 100];` (100 zeros) |
| Access | `a[0]`, `a[1]` (bracket notation, runtime index) |
| Same type | ✅ All elements must be same type |
| Size | Fixed at compile time (part of the type) |
| Bounds checking | ✅ Yes (panics on out-of-bounds) |
| Slices | `&a[1..4]` — type `&[T]` |
| Stored on | Stack |

### Key Takeaways

1. **Tuples** hold mixed types, **arrays** hold same types
2. Both have **fixed size** — use `Vec<T>` if you need dynamic sizing
3. Tuple access uses **dot notation** (`.0`), array access uses **brackets** (`[0]`)
4. Array length is **part of the type**: `[i32; 3]` ≠ `[i32; 5]`
5. Rust **checks array bounds** at runtime — no buffer overflows!
6. Use **slices** (`&[T]`) in function parameters for flexibility
7. Arrays live on the **stack** — fast but limited in size

---

## What's Next?

Now that we can store data, let's learn how to organize code into reusable pieces — functions!

**Next Tutorial:** [Functions →](./05-functions.md)

---

<p align="center">
  <i>Tutorial 4 of 8 — Stage 2: Rust Fundamentals</i>
</p>

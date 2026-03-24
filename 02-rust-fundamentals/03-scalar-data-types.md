# Scalar Data Types 🔢

> **Scalar types represent a single value. Rust has four scalar types: integers, floating-point numbers, booleans, and characters.**

---

## Table of Contents

- [What is a Scalar Type?](#what-is-a-scalar-type)
- [Integer Types — Whole Numbers](#integer-types--whole-numbers)
  - [Signed vs Unsigned](#signed-vs-unsigned)
  - [The Full Integer Table](#the-full-integer-table)
  - [How Integers Are Stored in Memory (Two's Complement)](#how-integers-are-stored-in-memory-twos-complement)
  - [Integer Literals — Writing Numbers in Code](#integer-literals--writing-numbers-in-code)
  - [Integer Overflow — What Happens When Numbers Get Too Big](#integer-overflow--what-happens-when-numbers-get-too-big)
  - [Type Suffixes](#type-suffixes)
  - [isize and usize — Platform-Dependent Sizes](#isize-and-usize--platform-dependent-sizes)
  - [Which Integer Type Should You Use?](#which-integer-type-should-you-use)
- [Floating-Point Types — Decimal Numbers](#floating-point-types--decimal-numbers)
  - [f32 vs f64](#f32-vs-f64)
  - [How Floats Work in Memory (IEEE 754)](#how-floats-work-in-memory-ieee-754)
  - [Floating-Point Gotchas](#floating-point-gotchas)
  - [Special Float Values](#special-float-values)
- [Numeric Operations](#numeric-operations)
- [The Boolean Type](#the-boolean-type)
- [The Character Type](#the-character-type)
- [Type Casting with as](#type-casting-with-as)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What is a Scalar Type?

A **scalar** type represents a **single value**. The term comes from mathematics and means "one-dimensional" — as opposed to composite/compound types (like arrays or structs) which hold multiple values.

Rust has **four** scalar types:

```
Scalar Types
├── Integer    → whole numbers (42, -7, 0)
├── Float      → decimal numbers (3.14, -0.001)
├── Boolean    → true or false
└── Character  → a single Unicode character ('A', '🦀', '中')
```

### The Origin of These Four Types

These four scalar types aren't arbitrary — they reflect fundamental categories that evolved over decades of programming language design:

- **Integers** are the oldest data type in computing. The earliest computers (1940s-50s) could ONLY work with integers. Floating-point hardware didn't become standard until the 1980s. Even today, integer arithmetic is the fastest operation a CPU performs — typically completing in a single clock cycle.

- **Floating-point numbers** were standardized in 1985 with **IEEE 754**, ending decades of chaos where every computer manufacturer had a different floating-point format. Before IEEE 754, the same program could give different results on different machines! William Kahan, the "Father of Floating Point," received the Turing Award for this work.

- **Booleans** are named after **George Boole** (1815-1864), a mathematician who invented Boolean algebra — the mathematical foundation of all digital logic. Every `if` statement, every `&&` and `||` operation, traces back to Boole's work, which predates computers by almost a century.

- **Characters** evolved from simple ASCII (128 characters, 1963) to Unicode (149,186+ characters, ongoing). The char type in Rust (4 bytes) can hold any Unicode scalar value, reflecting the modern reality that software must handle human languages from every culture.

Most languages have these same four categories, though they differ in the details. Rust's approach is notable for making the **size explicit** in the type name (`i32`, `u64`), which prevents an entire class of portability bugs that plague C programs (where `int` can be 16, 32, or 64 bits depending on the platform).

---

## Integer Types — Whole Numbers

Integers are numbers without a decimal point. Rust has **12 integer types** — that sounds like a lot, but each exists for a specific reason.

### Signed vs Unsigned

```
Signed (i):   Can be negative, zero, or positive  → -5, 0, 42
Unsigned (u): Can only be zero or positive          → 0, 42
```

The letter tells you:
- `i` = **i**nteger (signed — can be negative)
- `u` = **u**nsigned (cannot be negative)

The number tells you how many **bits** of storage:

```
i8   →  8 bits  = 1 byte
i16  → 16 bits  = 2 bytes
i32  → 32 bits  = 4 bytes    ← DEFAULT
i64  → 64 bits  = 8 bytes
i128 → 128 bits = 16 bytes
```

### The Full Integer Table

| Type | Size | Minimum Value | Maximum Value |
|------|------|--------------|---------------|
| `i8` | 1 byte | −128 | 127 |
| `i16` | 2 bytes | −32,768 | 32,767 |
| `i32` | 4 bytes | −2,147,483,648 | 2,147,483,647 |
| `i64` | 8 bytes | −9,223,372,036,854,775,808 | 9,223,372,036,854,775,807 |
| `i128` | 16 bytes | −170,141,183,460,469,231,731,687,303,715,884,105,728 | 170,141,183,460,469,231,731,687,303,715,884,105,727 |
| `u8` | 1 byte | 0 | 255 |
| `u16` | 2 bytes | 0 | 65,535 |
| `u32` | 4 bytes | 0 | 4,294,967,295 |
| `u64` | 8 bytes | 0 | 18,446,744,073,709,551,615 |
| `u128` | 16 bytes | 0 | 340,282,366,920,938,463,463,374,607,431,768,211,455 |
| `isize` | pointer size | Platform dependent | Platform dependent |
| `usize` | pointer size | 0 | Platform dependent |

### The Range Formula

For an N-bit type:
- **Signed:** $-(2^{N-1})$ to $2^{N-1} - 1$
- **Unsigned:** $0$ to $2^N - 1$

For example, `i8` (8 bits, signed):
- Min: $-(2^7) = -128$
- Max: $2^7 - 1 = 127$
- Total values: $2^8 = 256$ (from −128 to 127, inclusive)

For `u8` (8 bits, unsigned):
- Min: $0$
- Max: $2^8 - 1 = 255$
- Total values: $2^8 = 256$ (from 0 to 255)

### How Integers Are Stored in Memory (Two's Complement)

Let's see how the computer actually stores integers.

#### Unsigned Integers (Simple Binary)

An unsigned 8-bit integer (`u8`) is just a binary number:

```
Decimal:  42
Binary:   0 0 1 0 1 0 1 0
          │ │ │ │ │ │ │ │
          │ │ │ │ │ │ │ └── 2⁰ = 0
          │ │ │ │ │ │ └──── 2¹ = 2
          │ │ │ │ │ └────── 2² = 0
          │ │ │ │ └──────── 2³ = 8
          │ │ │ └────────── 2⁴ = 0
          │ │ └──────────── 2⁵ = 32
          │ └────────────── 2⁶ = 0
          └──────────────── 2⁷ = 0
                            ────
                    Sum:     42  ✓
```

Range of `u8`: 00000000 (0) to 11111111 (255).

#### Signed Integers (Two's Complement)

Signed integers use **two's complement** encoding. The leftmost bit (highest bit) is the **sign bit**:

```
If the highest bit is 0 → the number is POSITIVE (or zero)
If the highest bit is 1 → the number is NEGATIVE
```

Examples for `i8`:
```
 0000_0000 =    0
 0000_0001 =    1
 0000_0010 =    2
 0111_1111 =  127   (maximum positive value)
 1000_0000 = -128   (minimum negative value)
 1111_1111 =   -1
 1111_1110 =   -2
```

To find the two's complement of a negative number:
1. Write the positive version in binary
2. Flip all the bits (0→1, 1→0)
3. Add 1

Example: Convert -42 to `i8`:
```
Step 1:  42  = 0010_1010
Step 2: flip = 1101_0101
Step 3:  +1  = 1101_0110  ← This is -42 in two's complement
```

> **Why two's complement?** Because the CPU can use the same circuitry for addition regardless of whether numbers are positive or negative. The hardware doesn't need to know if numbers are signed!

### The History of Negative Number Representation

Two's complement wasn't always the standard. Early computers used different systems:

**Sign-magnitude (1940s-50s):** The simplest idea — use one bit for the sign, the rest for the value:
```
+5 = 0_000_0101
-5 = 1_000_0101  (just flip the sign bit)
```
Problem: There are TWO representations of zero (+0 and -0), and the hardware for addition is complex because it needs to check the sign bit first.

**One's complement (1950s-60s):** Negate by flipping ALL bits:
```
+5 = 0000_0101
-5 = 1111_1010  (all bits flipped)
```
Still has two zeros (+0 = 00000000, -0 = 11111111), and addition requires an "end-around carry."

**Two's complement (1960s-present):** The winner. Only ONE zero, and the same addition circuitry works for both positive and negative:
```
+5 = 0000_0101
-5 = 1111_1011  (flip all bits, add 1)
 0 = 0000_0000  (only ONE zero!)
```

Two's complement won because it simplifies hardware design — the CPU's Arithmetic Logic Unit (ALU) doesn't need separate circuits for signed and unsigned addition. This single design decision saved billions of transistors across the history of computing.

> **Real-World Impact:** Every modern CPU (x86, ARM, RISC-V) uses two's complement. It's not just a convention — it's hardwired into the silicon. When Rust specifies `i8`, `i16`, `i32`, etc., it's directly mapping to how the CPU represents these numbers in its registers.

### Checking the Range in Code

```rust
fn main() {
    println!("i8:  {} to {}", i8::MIN, i8::MAX);
    println!("i16: {} to {}", i16::MIN, i16::MAX);
    println!("i32: {} to {}", i32::MIN, i32::MAX);
    println!("i64: {} to {}", i64::MIN, i64::MAX);
    println!("u8:  {} to {}", u8::MIN, u8::MAX);
    println!("u16: {} to {}", u16::MIN, u16::MAX);
    println!("u32: {} to {}", u32::MIN, u32::MAX);
    println!("u64: {} to {}", u64::MIN, u64::MAX);
}
```

Output:
```
i8:  -128 to 127
i16: -32768 to 32767
i32: -2147483648 to 2147483647
i64: -9223372036854775808 to 9223372036854775807
u8:  0 to 255
u16: 0 to 65535
u32: 0 to 4294967295
u64: 0 to 18446744073709551615
```

### Integer Literals — Writing Numbers in Code

Rust lets you write integers in several ways:

```rust
fn main() {
    // Decimal (base 10) — most common
    let decimal = 98_222;      // Underscores are visual separators
    
    // Hexadecimal (base 16) — prefix: 0x
    let hex = 0xFF;            // = 255
    let hex_color = 0x1A2B3C;  // = 1,715,004
    
    // Octal (base 8) — prefix: 0o
    let octal = 0o77;          // = 63
    let permissions = 0o755;   // Unix file permissions
    
    // Binary (base 2) — prefix: 0b
    let binary = 0b1111_0000;  // = 240
    let flags = 0b0000_0101;   // Bit flags
    
    // Byte (u8 only) — prefix: b
    let byte = b'A';           // = 65 (ASCII value of 'A')
    
    println!("decimal: {}", decimal);
    println!("hex: {}", hex);
    println!("octal: {}", octal);
    println!("binary: {}", binary);
    println!("byte: {}", byte);
}
```

Output:
```
decimal: 98222
hex: 255
octal: 63
binary: 240
byte: 65
```

### Visual Separators with Underscores

Underscores in number literals are purely visual — they're ignored by the compiler:

```rust
let million = 1_000_000;       // Easy to read: one million
let credit_card = 4111_1111_1111_1111_u64;  // Groups of 4
let binary = 0b1111_0000_1010_0101;  // Groups of 4 bits
let hex = 0xFF_FF_FF;          // Groups of 2
```

> **Real-World Practice:** Use underscores for numbers above 10,000. It prevents costly bugs from miscounting zeros (Is that 1,000,000 or 10,000,000?).

### Integer Overflow — What Happens When Numbers Get Too Big

What happens if you try to store 256 in a `u8` (max is 255)?

```rust
fn main() {
    let x: u8 = 255;
    let y: u8 = x + 1;  // What happens?
}
```

**Behavior depends on the build mode:**

| Build Mode | Behavior | Example (255 + 1 in u8) |
|------------|----------|------------------------|
| Debug (`cargo build`) | **Panics** (crashes) | Program crashes with "overflow" error |
| Release (`cargo build --release`) | **Wraps around** | 255 + 1 = 0 |

```
Wrapping (in release mode):

u8 values go from 0 to 255.
255 + 1 wraps to 0
255 + 2 wraps to 1
0 - 1 wraps to 255

Think of it like a clock:
  0 → 1 → 2 → ... → 254 → 255 → 0 → 1 → ...
```

### Explicit Overflow Handling

Rust provides methods to handle overflow explicitly:

```rust
fn main() {
    let x: u8 = 250;
    
    // 1. wrapping_add: always wraps
    let a = x.wrapping_add(10);           // 250 + 10 = 4 (wraps)
    println!("wrapping: {}", a);
    
    // 2. checked_add: returns None on overflow
    let b = x.checked_add(10);            // None (overflow)
    let c = x.checked_add(5);             // Some(255)
    println!("checked overflow: {:?}", b);
    println!("checked ok: {:?}", c);
    
    // 3. saturating_add: clamps to max
    let d = x.saturating_add(10);         // 255 (clamped to max)
    println!("saturating: {}", d);
    
    // 4. overflowing_add: returns (value, did_overflow)
    let (e, overflowed) = x.overflowing_add(10);
    println!("overflowing: {} (overflow: {})", e, overflowed);
    // overflowing: 4 (overflow: true)
}
```

> **Real-World Practice:** Use `checked_*` methods when overflow is a genuine possibility (user input, untrusted data). Use `wrapping_*` when you intentionally want wrapping behavior (hash functions, circular buffers). Use `saturating_*` for clamping (audio, pixel colors).

### Type Suffixes

You can add the type directly to a number literal:

```rust
fn main() {
    let a = 42i32;     // i32
    let b = 42_i32;    // i32 (underscore for readability)
    let c = 42u8;      // u8
    let d = 42_u64;    // u64
    let e = 1_000i64;  // i64
}
```

This is especially useful when calling methods or when type inference needs help:

```rust
fn main() {
    // Without suffix, Rust defaults to i32. But .pow() needs a u32 exponent:
    println!("{}", 2_i64.pow(10));  // 1024

    // In a generic context where type matters:
    let bytes = 1024_u64 * 1024 * 1024;  // 1 GB — needs u64 to not overflow
}
```

### `isize` and `usize` — Platform-Dependent Sizes

These two special types match the **pointer size** of your platform:

```
On a 64-bit system:
  isize = i64 (8 bytes)
  usize = u64 (8 bytes)

On a 32-bit system:
  isize = i32 (4 bytes)
  usize = u32 (4 bytes)
```

#### When is `usize` required?

```rust
fn main() {
    let arr = [10, 20, 30, 40, 50];
    
    // Array indexing REQUIRES usize
    let index: usize = 2;
    println!("{}", arr[index]);  // 30
    
    // This won't work:
    // let i: i32 = 2;
    // println!("{}", arr[i]);  // ❌ ERROR: expected usize, found i32
    
    // .len() returns usize
    let length: usize = arr.len();
    println!("Length: {}", length);  // 5
}
```

#### Why `usize` for indexing?

Indexing is fundamentally a **memory address offset**. The size of a memory address depends on the platform (32-bit or 64-bit), so the index type must also be platform-dependent. Using `usize` guarantees:

1. The index can address any memory location on the platform
2. The index is never negative (arrays can't have negative indices in Rust)

> **Real-World Practice:** Use `usize` for collection sizes and indices. Use `i32` or `i64` for general-purpose integers. Use `u8` for byte-level data. Use `u32` for IDs and counters.

### Which Integer Type Should You Use?

| Situation | Recommended Type | Why |
|-----------|-----------------|-----|
| General integers | `i32` | Default, fast on all platforms |
| Array/collection indices | `usize` | Required by Rust |
| Positive counts/sizes | `usize` or `u32` | Can't be negative |
| Byte data | `u8` | Exactly one byte |
| Small values (0-255) | `u8` | Memory efficient |
| Pixel colors (RGBA) | `u8` | 0-255 range |
| Port numbers | `u16` | 0-65535 |
| Large numbers | `i64` / `u64` | File sizes, timestamps |
| Very large numbers | `i128` / `u128` | Cryptography, UUIDs |
| Matching C types | See `std::os::raw` | FFI interop |

---

## Floating-Point Types — Decimal Numbers

Floating-point numbers represent numbers with a decimal point:

```rust
fn main() {
    let x = 2.0;      // f64 (default)
    let y: f32 = 3.0;  // f32 (must be annotated)
    
    println!("x = {}, y = {}", x, y);
}
```

### `f32` vs `f64`

| Feature | `f32` | `f64` |
|---------|-------|-------|
| Size | 4 bytes (32 bits) | 8 bytes (64 bits) |
| Precision | ~7 decimal digits | ~15 decimal digits |
| Range | ±3.4 × 10³⁸ | ±1.8 × 10³⁰⁸ |
| Default? | No | **Yes** |
| Speed | Faster on some hardware | Same speed on modern 64-bit CPUs |

```rust
fn main() {
    // f64 has more precision
    let pi_f32: f32 = 3.14159265358979323846;
    let pi_f64: f64 = 3.14159265358979323846;
    
    println!("f32 pi: {:.20}", pi_f32);  // 3.14159274101257324219
    println!("f64 pi: {:.20}", pi_f64);  // 3.14159265358979311600
    //                                       3.14159265358979323846  ← actual
    // Notice f32 diverges after 7 digits, f64 after 15 digits
}
```

> **Real-World Practice:** Use `f64` unless you have a specific reason for `f32` (GPU programming, large arrays where memory matters, matching C `float`). On modern 64-bit CPUs, `f64` is the same speed as `f32`.

### How Floats Work in Memory (IEEE 754)

Floating-point numbers follow the **IEEE 754** standard. A `f64` is stored as:

```
┌─┬───────────┬────────────────────────────────────────────────────┐
│S│ Exponent  │                  Fraction (Mantissa)                │
│1│  11 bits  │                      52 bits                        │
└─┴───────────┴────────────────────────────────────────────────────┘

S = Sign bit (0 = positive, 1 = negative)
```

The value is calculated as:

$$(-1)^S \times 2^{E-1023} \times 1.F$$

Where:
- S = sign bit
- E = exponent (11 bits)
- F = fraction (52 bits)

For `f32`:
```
┌─┬──────────┬───────────────────────┐
│S│ Exponent │   Fraction (Mantissa)  │
│1│  8 bits  │        23 bits         │
└─┴──────────┴───────────────────────┘
```

### A Real-World Example: How 3.14 is Stored

Let's trace how the number 3.14 gets stored as an `f64`:

1. **Convert to binary:** 3.14 in binary ≈ 11.001000111101011100001... (repeating)
2. **Normalize:** Move the decimal point: 1.1001000111101011100001... × 2¹
3. **Encode:**
   - Sign bit: 0 (positive)
   - Exponent: 1 + 1023 (bias) = 1024 = 10000000000 in binary
   - Mantissa: 1001000111101011100001... (52 bits, the leading 1 is implicit)

```
f64 representation of 3.14:
0 10000000000 1001000111101011100001010001111010111000010100011111
S  Exponent            Mantissa (52 bits)
```

The **bias** (1023 for f64, 127 for f32) exists so that exponents can represent both very large and very small numbers without needing a separate sign bit for the exponent.

### Why "Floating Point"?

The name comes from the fact that the decimal point "floats" — it can be positioned anywhere by changing the exponent:

```
1.5     = 1.1  × 2⁰     (point after first digit)
0.75    = 1.1  × 2⁻¹    (point "floats" left)
6.0     = 1.1  × 2²     (point "floats" right)
0.001   ≈ 1.0  × 2⁻¹⁰   (point far to the left)
```

This is in contrast to **fixed-point** numbers, where the decimal point is always in the same position (used in some embedded systems and financial calculations where you need exact decimal precision).

> **Historical Note:** Before IEEE 754 (1985), floating-point arithmetic was a mess. Different computers used different formats — IBM had its own hexadecimal floating-point, DEC had its own format, and Cray supercomputers had yet another. Programs would give different answers on different machines! IEEE 754 standardized everything, and now every CPU in the world uses the same floating-point format.

You don't need to memorize this, but understanding it explains WHY floating-point math has precision issues.

### Floating-Point Gotchas

#### Gotcha 1: Not All Numbers Can Be Represented Exactly

```rust
fn main() {
    let a: f64 = 0.1;
    let b: f64 = 0.2;
    let c = a + b;
    
    println!("0.1 + 0.2 = {:.20}", c);
    // 0.1 + 0.2 = 0.30000000000000004441
    
    // This is NOT a Rust bug — it's how ALL programming languages handle floats
    // (C, Java, Python, JavaScript — they all have this issue)
    
    // ❌ NEVER compare floats with ==
    if c == 0.3 {
        println!("Equal!");     // This will NOT print!
    }
    
    // ✅ Compare with a small epsilon
    let epsilon = 1e-10;
    if (c - 0.3).abs() < epsilon {
        println!("Close enough!");  // This WILL print
    }
}
```

Why? Because 0.1 in binary is a repeating fraction (like 1/3 = 0.333... in decimal):

```
0.1 in decimal = 0.0001100110011001100... in binary (repeats forever)
```

Since we only have 52 bits of precision, we round, which introduces a tiny error.

#### Gotcha 2: Precision Loss with Large Numbers

```rust
fn main() {
    let big: f64 = 1_000_000_000_000_000.0;  // 10^15
    let big_plus_one = big + 1.0;
    
    println!("{}", big_plus_one - big);  // 1 ✓ (still fine)
    
    let very_big: f64 = 1e18;  // 10^18
    let very_big_plus_one = very_big + 1.0;
    
    println!("{}", very_big_plus_one - very_big);  // 0 ✗ (precision lost!)
}
```

When numbers get very large, floating-point types can't represent individual integers anymore. This is because the mantissa (52 bits for f64) can only hold so many significant digits.

#### Gotcha 3: Floats Cannot Be Used as HashMap Keys

```rust
// ❌ This won't compile
// let map: HashMap<f64, String> = HashMap::new();
// Because: f64 does not implement Eq or Hash
// Reason: NaN != NaN, so equality is undefined for floats
```

### Special Float Values

```rust
fn main() {
    // Infinity
    let inf = f64::INFINITY;
    let neg_inf = f64::NEG_INFINITY;
    println!("1.0 / 0.0 = {}", 1.0_f64 / 0.0);  // inf
    
    // NaN (Not a Number)
    let nan = f64::NAN;
    println!("sqrt(-1) = {}", (-1.0_f64).sqrt());  // NaN
    
    // NaN is weird — it's not equal to anything, including itself!
    println!("NaN == NaN: {}", nan == nan);          // false!
    println!("NaN != NaN: {}", nan != nan);          // true!
    
    // Check for NaN with .is_nan()
    println!("Is NaN: {}", nan.is_nan());            // true
    
    // Useful constants
    println!("PI: {}", std::f64::consts::PI);
    println!("E:  {}", std::f64::consts::E);
    println!("Max f64: {}", f64::MAX);
    println!("Min positive f64: {}", f64::MIN_POSITIVE);
}
```

---

## Numeric Operations

Rust supports all standard arithmetic operations:

```rust
fn main() {
    // Addition
    let sum = 5 + 10;
    println!("5 + 10 = {}", sum);        // 15
    
    // Subtraction
    let difference = 95.5 - 4.3;
    println!("95.5 - 4.3 = {}", difference);  // 91.2
    
    // Multiplication
    let product = 4 * 30;
    println!("4 * 30 = {}", product);    // 120
    
    // Division
    let quotient = 56.7 / 32.2;
    println!("56.7 / 32.2 = {}", quotient);  // 1.7608...
    
    // Integer division (truncates toward zero)
    let truncated = -5 / 3;
    println!("-5 / 3 = {}", truncated);  // -1 (not -2!)
    
    // Remainder (modulo)
    let remainder = 43 % 5;
    println!("43 % 5 = {}", remainder);  // 3
    
    // Negation
    let negative = -42;
    println!("negative: {}", negative);
}
```

### Integer Division — Important Detail!

```rust
fn main() {
    // Integer division TRUNCATES (removes the decimal, rounds toward zero)
    println!("7 / 2 = {}", 7 / 2);          // 3 (not 3.5!)
    println!("-7 / 2 = {}", -7 / 2);        // -3 (not -4!)
    println!("1 / 3 = {}", 1 / 3);          // 0
    
    // To get decimal results, use floats:
    println!("7.0 / 2.0 = {}", 7.0 / 2.0);  // 3.5
    
    // Cannot mix integers and floats:
    // println!("{}", 7 / 2.0);  // ❌ ERROR: cannot divide i32 by f64
    
    // Must convert explicitly:
    println!("{}", 7.0 / 2.0);              // 3.5
    println!("{}", 7 as f64 / 2 as f64);    // 3.5
}
```

### Compound Assignment Operators

```rust
fn main() {
    let mut x = 10;
    
    x += 5;   // x = x + 5  → 15
    x -= 3;   // x = x - 3  → 12
    x *= 2;   // x = x * 2  → 24
    x /= 4;   // x = x / 4  → 6
    x %= 4;   // x = x % 4  → 2
    
    println!("Final x: {}", x);  // 2
}
```

### Bitwise Operations

```rust
fn main() {
    let a: u8 = 0b1010;  // 10
    let b: u8 = 0b1100;  // 12
    
    println!("AND:  {:08b}", a & b);   // 00001000 (8)
    println!("OR:   {:08b}", a | b);   // 00001110 (14)
    println!("XOR:  {:08b}", a ^ b);   // 00000110 (6)
    println!("NOT:  {:08b}", !a);      // 11110101 (245 for u8)
    println!("SHL:  {:08b}", a << 2);  // 00101000 (40)
    println!("SHR:  {:08b}", a >> 1);  // 00000101 (5)
}
```

### You Cannot Mix Types in Operations

```rust
fn main() {
    let a: i32 = 5;
    let b: i64 = 10;
    
    // let c = a + b;  // ❌ ERROR: cannot add i32 to i64
    let c = a as i64 + b;  // ✅ Convert first, then add
    
    let x: i32 = 5;
    let y: f64 = 2.0;
    
    // let z = x + y;  // ❌ ERROR: cannot add i32 to f64
    let z = x as f64 + y;  // ✅ Convert first
    
    println!("c = {}, z = {}", c, z);
}
```

Rust forces explicit type conversions because implicit conversions can silently lose data:

```
In C:   int x = 300; char y = x;  // y is 44! (silently truncated, 300 - 256 = 44)
In Rust: let x: i32 = 300; let y: u8 = x;  // ❌ COMPILE ERROR — you must be explicit
```

---

## The Boolean Type

The `bool` type has exactly two values: `true` and `false`.

```rust
fn main() {
    let is_active: bool = true;
    let is_deleted = false;   // Type inference works
    
    println!("Active: {}", is_active);   // Active: true
    println!("Deleted: {}", is_deleted); // Deleted: false
}
```

### Size in Memory

A `bool` takes **1 byte** (8 bits), even though it only needs 1 bit. Why?

```
The smallest addressable unit of memory is a BYTE (8 bits).
You can't have a pointer to a single bit.
So Rust uses a full byte:

  true  = 0x01 = 00000001
  false = 0x00 = 00000000
```

### Boolean Operations

```rust
fn main() {
    let a = true;
    let b = false;
    
    // Logical AND — both must be true
    println!("true AND false: {}", a && b);   // false
    println!("true AND true:  {}", a && a);   // true
    
    // Logical OR — at least one must be true
    println!("true OR false:  {}", a || b);   // true
    println!("false OR false: {}", b || b);   // false
    
    // Logical NOT — flips the value
    println!("NOT true:  {}", !a);            // false
    println!("NOT false: {}", !b);            // true
}
```

### Truth Table

```
 A     B     A && B   A || B   !A
─────────────────────────────────
true  true   true     true     false
true  false  false    true     false
false true   false    true     true
false false  false    false    true
```

### Short-Circuit Evaluation

`&&` and `||` are **short-circuit** operators — they stop evaluating as soon as the result is determined:

```rust
fn expensive_check() -> bool {
    println!("  (expensive check ran!)");
    true
}

fn main() {
    // AND: if left side is false, right side is NEVER evaluated
    println!("false && expensive_check():");
    let result = false && expensive_check();
    println!("  Result: {}\n", result);
    // Output: only "Result: false" — expensive_check() never runs!
    
    // OR: if left side is true, right side is NEVER evaluated
    println!("true || expensive_check():");
    let result = true || expensive_check();
    println!("  Result: {}", result);
    // Output: only "Result: true" — expensive_check() never runs!
}
```

> **Real-World Practice:** Short-circuit evaluation is used for guard conditions:
> ```rust
> if !list.is_empty() && list[0] == target { ... }
> // If list is empty, list[0] is NEVER accessed (prevents crash!)
> ```

### Booleans from Comparisons

```rust
fn main() {
    let x = 10;
    let y = 20;
    
    println!("{} == {}: {}", x, y, x == y);  // false (equal)
    println!("{} != {}: {}", x, y, x != y);  // true  (not equal)
    println!("{} <  {}: {}", x, y, x < y);   // true  (less than)
    println!("{} >  {}: {}", x, y, x > y);   // false (greater than)
    println!("{} <= {}: {}", x, y, x <= y);   // true  (less or equal)
    println!("{} >= {}: {}", x, y, x >= y);   // false (greater or equal)
}
```

---

## The Character Type

Rust's `char` type represents a single **Unicode scalar value**. It's NOT just ASCII!

```rust
fn main() {
    let c = 'z';
    let z: char = 'ℤ';
    let heart_cat = '😻';
    let chinese = '中';
    let hindi = 'ह';
    let musical = '🎵';
    let rust = '🦀';
    
    println!("{} {} {} {} {} {} {}", c, z, heart_cat, chinese, hindi, musical, rust);
}
```

### Size in Memory — 4 Bytes!

A `char` in Rust is **4 bytes** (32 bits), NOT 1 byte like in C:

```
C:      char = 1 byte  = ASCII only (0-127)
Java:   char = 2 bytes = UTF-16 (some emoji need 2 chars!)
Rust:   char = 4 bytes = Any Unicode scalar value

Unicode scalar values: U+0000 to U+D7FF and U+E000 to U+10FFFF
Total possible values: 1,112,064
```

Why 4 bytes? Because a single Unicode code point can require up to 21 bits, and 4 bytes (32 bits) is the smallest standard size that fits.

### Important: `char` vs String

```rust
fn main() {
    let c: char = 'A';         // Single quotes → char (always ONE character)
    let s: &str = "A";         // Double quotes → string slice (could be any length)
    let s2: String = String::from("A");  // Owned string
    
    // char is 4 bytes, but in a string, 'A' is only 1 byte (UTF-8 encoding)
    println!("char size: {} bytes", std::mem::size_of::<char>());     // 4
    println!("&str 'A' len: {} bytes", "A".len());                     // 1
    println!("&str '中' len: {} bytes", "中".len());                    // 3
    println!("&str '🦀' len: {} bytes", "🦀".len());                   // 4
}
```

### Character Methods

```rust
fn main() {
    let c = 'A';
    
    println!("Is alphabetic: {}", c.is_alphabetic());    // true
    println!("Is numeric: {}", c.is_numeric());          // false
    println!("Is alphanumeric: {}", c.is_alphanumeric());// true
    println!("Is uppercase: {}", c.is_uppercase());      // true
    println!("Is lowercase: {}", c.is_lowercase());      // false
    println!("Is whitespace: {}", ' '.is_whitespace());  // true
    println!("To lowercase: {}", c.to_lowercase().next().unwrap()); // 'a'
    println!("To uppercase: {}", 'a'.to_uppercase().next().unwrap()); // 'A'
    
    // Unicode code point
    println!("'A' = U+{:04X}", c as u32);     // U+0041
    println!("'🦀' = U+{:04X}", '🦀' as u32); // U+1F980
    
    // Digit value
    println!("'7' as digit: {:?}", '7'.to_digit(10));   // Some(7)
    println!("'F' as hex:   {:?}", 'F'.to_digit(16));   // Some(15)
}
```

### Char Literals with Escape Sequences

```rust
fn main() {
    let newline = '\n';       // Newline
    let tab = '\t';           // Tab
    let backslash = '\\';     // Backslash
    let single_quote = '\'';  // Single quote
    let null = '\0';          // Null character
    
    // Unicode escape
    let heart = '\u{2764}';   // ❤
    let rust = '\u{1F980}';   // 🦀
    
    println!("Heart: {}, Crab: {}", heart, rust);
}
```

---

## Type Casting with `as`

Rust requires **explicit** type conversions. The simplest way is the `as` keyword:

```rust
fn main() {
    // Integer to integer
    let x: i32 = 42;
    let y: i64 = x as i64;      // Safe: wider type
    let z: i16 = x as i16;      // Possible truncation!
    
    // Integer to float
    let a: i32 = 42;
    let b: f64 = a as f64;      // 42.0
    
    // Float to integer (TRUNCATES — does NOT round!)
    let c: f64 = 3.99;
    let d: i32 = c as i32;      // 3 (not 4! Truncated toward zero!)
    
    // Float to float
    let e: f64 = 3.14;
    let f: f32 = e as f32;      // Possible precision loss
    
    // bool to integer
    let g: bool = true;
    let h: i32 = g as i32;      // 1
    let i: i32 = false as i32;  // 0
    
    // char to integer (Unicode code point)
    let j: char = 'A';
    let k: u32 = j as u32;      // 65
    
    // Integer to char (only u8 can convert directly)
    let l: u8 = 65;
    let m: char = l as char;    // 'A'
    
    println!("f64 → i32: {} → {}", c, d);    // 3.99 → 3
    println!("bool → i32: true → {}", h);     // true → 1
    println!("char → u32: 'A' → {}", k);      // 'A' → 65
}
```

### Dangers of `as` Casting

```rust
fn main() {
    // Narrowing can silently truncate!
    let big: i32 = 300;
    let small: u8 = big as u8;
    println!("300 as u8 = {}", small);  // 44 (300 - 256 = 44, wraps around!)
    
    // Negative to unsigned — wraps around
    let negative: i32 = -1;
    let unsigned: u32 = negative as u32;
    println!("-1 as u32 = {}", unsigned);  // 4294967295 (u32::MAX)
    
    // Float to integer — special cases
    let nan: f64 = f64::NAN;
    let inf: f64 = f64::INFINITY;
    println!("NaN as i32 = {}", nan as i32);      // 0
    println!("Infinity as i32 = {}", inf as i32);  // 2147483647 (i32::MAX)
}
```

> **Real-World Practice:** The `as` keyword is quick but dangerous for narrowing conversions. For safe conversions, use the `TryFrom` / `TryInto` traits (covered in Stage 11):
> ```rust
> let big: i32 = 300;
> let small: Result<u8, _> = u8::try_from(big);  // Err(TryFromIntError)
> ```

---

## Common Mistakes

### Mistake 1: Mixing Integer Types

```rust
let a: i32 = 5;
let b: u32 = 10;
// let c = a + b;  // ❌ ERROR: cannot add i32 to u32
let c = a + b as i32;  // ✅ Convert first
```

### Mistake 2: Integer Division When You Want Float Division

```rust
let result = 5 / 2;        // 2 (not 2.5!)
let result = 5.0 / 2.0;    // 2.5 ✅
let result = 5 as f64 / 2 as f64;  // 2.5 ✅
```

### Mistake 3: Comparing Floats with ==

```rust
// ❌ WRONG
if 0.1 + 0.2 == 0.3 { println!("equal"); }  // Will NOT print!

// ✅ CORRECT
if (0.1_f64 + 0.2 - 0.3).abs() < f64::EPSILON { println!("close enough"); }
```

### Mistake 4: Using a Negative Literal for Unsigned

```rust
// let x: u32 = -5;  // ❌ ERROR: cannot apply unary operator `-` to type `u32`
let x: i32 = -5;     // ✅ Use signed type for negative numbers
```

### Mistake 5: Forgetting char is Unicode, Not ASCII

```rust
let c: char = 'A';
println!("{}", c as u8);  // 65 ✅
// let c2: char = '🦀';
// println!("{}", c2 as u8);  // ❌ Will truncate! 🦀 doesn't fit in u8
```

---

## Exercises

### Exercise 1: Memory Sizes

Print the size in bytes of every scalar type (`i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`, `u128`, `f32`, `f64`, `bool`, `char`) using `std::mem::size_of::<Type>()`.

### Exercise 2: Integer Ranges

Write a program that prints the minimum and maximum values for `i8`, `i16`, `i32`, `u8`, `u16`, `u32`.

### Exercise 3: Temperature Converter

Write a program that converts temperatures. Store a temperature as `f64`, convert it from Fahrenheit to Celsius and back, then print both.

Formulas:
- $C = (F - 32) \times \frac{5}{9}$
- $F = C \times \frac{9}{5} + 32$

### Exercise 4: Overflow Exploration

Using a `u8` variable with value `250`, try these operations and print the results:
- `wrapping_add(10)`
- `checked_add(10)`
- `saturating_add(10)`
- `overflowing_add(10)`

### Exercise 5: Character Detective

Write a program that takes the character `'7'` and:
1. Prints whether it's alphabetic, numeric, or alphanumeric
2. Converts it to its digit value (using `.to_digit(10)`)
3. Prints its Unicode code point

### Exercise 6: Type Casting Chain

Start with the `f64` value `3.99` and cast it through this chain, printing at each step:
`f64 → i32 → u8 → char`

What character do you get at the end?

---

## Summary

### Integer Types

| Type | Size | Range |
|------|------|-------|
| `i8` / `u8` | 1 byte | −128…127 / 0…255 |
| `i16` / `u16` | 2 bytes | −32K…32K / 0…65K |
| `i32` / `u32` | 4 bytes | −2.1B…2.1B / 0…4.2B |
| `i64` / `u64` | 8 bytes | Very large |
| `i128` / `u128` | 16 bytes | Extremely large |
| `isize` / `usize` | Pointer-sized | Platform dependent |

### Float Types

| Type | Size | Precision | Default? |
|------|------|-----------|----------|
| `f32` | 4 bytes | ~7 digits | No |
| `f64` | 8 bytes | ~15 digits | **Yes** |

### Other Scalars

| Type | Size | Values |
|------|------|--------|
| `bool` | 1 byte | `true`, `false` |
| `char` | 4 bytes | Any Unicode scalar value |

### Key Takeaways

1. **Default integer: `i32`** — fast, sufficient for most uses
2. **Default float: `f64`** — same speed as f32 on modern CPUs, more precise
3. **Rust never implicitly converts types** — you must use `as` or conversion methods
4. **Integer overflow panics in debug, wraps in release** — use `checked_*` methods for safety
5. **Never compare floats with `==`** — use epsilon comparisons
6. **`char` is 4 bytes** (Unicode), not 1 byte (ASCII)
7. **`usize` is required** for array indexing and collection sizes

---

## What's Next?

Now that we know the single-value types, let's look at types that hold MULTIPLE values — tuples and arrays.

**Next Tutorial:** [Compound Data Types →](./04-compound-data-types.md)

---

<p align="center">
  <i>Tutorial 3 of 8 — Stage 2: Rust Fundamentals</i>
</p>

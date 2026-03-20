# String vs &str 🔤

> **`String` and `&str` are two different types that both represent text. Understanding when to use each is one of the most important practical skills in Rust. This tutorial covers everything you need to know.**

---

## Table of Contents

- [Why Two String Types?](#why-two-string-types)
- [String — The Owned String](#string--the-owned-string)
  - [What String Looks Like in Memory](#what-string-looks-like-in-memory)
  - [Creating Strings](#creating-strings)
  - [Modifying Strings](#modifying-strings)
  - [String Growth and Capacity](#string-growth-and-capacity)
- [&str — The Borrowed String Slice](#str--the-borrowed-string-slice)
  - [What &str Looks Like in Memory](#what-str-looks-like-in-memory)
  - [Where &str Comes From](#where-str-comes-from)
- [Side-by-Side Comparison](#side-by-side-comparison)
- [When to Use Which](#when-to-use-which)
  - [Use &str When...](#use-str-when)
  - [Use String When...](#use-string-when)
  - [Function Parameters: The Golden Rule](#function-parameters-the-golden-rule)
  - [Function Return Values](#function-return-values)
  - [Struct Fields](#struct-fields)
- [Converting Between String and &str](#converting-between-string-and-str)
  - [String → &str (Cheap)](#string--str-cheap)
  - [&str → String (Allocates)](#str--string-allocates)
- [Deref Coercion](#deref-coercion)
- [String Internals: UTF-8](#string-internals-utf-8)
  - [Indexing Strings](#indexing-strings)
  - [Iterating Over Characters](#iterating-over-characters)
  - [Getting the Nth Character](#getting-the-nth-character)
- [Common String Operations](#common-string-operations)
- [to_string() vs to_owned() vs String::from()](#to_string-vs-to_owned-vs-stringfrom)
- [&String vs &str](#string-vs-str-1)
- [Strings in Collections](#strings-in-collections)
- [Real-World Patterns](#real-world-patterns)
- [Common Mistakes](#common-mistakes)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Why Two String Types?

In languages like Python or JavaScript, there's one string type. Rust has two because of ownership:

```
Python:  s = "hello"    # One type, garbage collected
Java:    String s = "hello"  # One type, garbage collected
C:       char *s = "hello"   # Raw pointer, manual management
Rust:    Two types because ownership matters:
         - String    → owned, heap-allocated, growable
         - &str      → borrowed, a view into string data
```

The split is about **who owns the data** and **where it lives**:

```
┌───────────────────────────────────────────────────┐
│  String                                           │
│  • Owns its data                                  │
│  • Data lives on the heap                         │
│  • Can grow, shrink, modify                       │
│  • Dropped when owner goes out of scope           │
│  • Like Vec<u8> but guaranteed valid UTF-8         │
├───────────────────────────────────────────────────┤
│  &str                                             │
│  • Borrows data (doesn't own it)                  │
│  • Points to data anywhere (heap, stack, binary)  │
│  • Read-only (unless &mut str)                    │
│  • Size = pointer + length (16 bytes on stack)    │
│  • Like &[u8] but guaranteed valid UTF-8          │
└───────────────────────────────────────────────────┘
```

---

## String — The Owned String

A `String` is an **owned, heap-allocated, growable** UTF-8 string. It's very similar to `Vec<u8>` with a UTF-8 guarantee.

### What String Looks Like in Memory

```rust
let s = String::from("hello");
```

```
Stack:                    Heap:
┌────────────┐            ┌───┬───┬───┬───┬───┐
│ ptr ───────────────────→│ h │ e │ l │ l │ o │
│ len: 5     │            └───┴───┴───┴───┴───┘
│ capacity: 5│              (5 bytes of UTF-8 data)
└────────────┘
  24 bytes on stack         variable bytes on heap
  (3 × 8 bytes = ptr + len + capacity)
```

Three fields on the stack:
- **ptr**: pointer to heap-allocated bytes
- **len**: how many bytes are currently used
- **capacity**: how many bytes are allocated (always ≥ len)

### Creating Strings

```rust
fn main() {
    // From a string literal
    let s1 = String::from("hello");
    let s2 = "hello".to_string();
    let s3 = "hello".to_owned();
    
    // Empty string
    let s4 = String::new();
    
    // With pre-allocated capacity (avoids reallocation)
    let s5 = String::with_capacity(100);
    
    // From other types
    let s6 = 42.to_string();           // "42"
    let s7 = format!("{} {}", "hello", "world");  // "hello world"
    let s8 = ['h', 'i'].iter().collect::<String>(); // "hi"
    
    // From bytes (unsafe if not valid UTF-8)
    let bytes = vec![104, 101, 108, 108, 111];
    let s9 = String::from_utf8(bytes).unwrap();  // "hello"
}
```

### Modifying Strings

Since `String` is owned, you can modify it:

```rust
fn main() {
    let mut s = String::from("hello");
    
    // Append a string slice
    s.push_str(" world");
    println!("{}", s);  // "hello world"
    
    // Append a single character
    s.push('!');
    println!("{}", s);  // "hello world!"
    
    // Insert at position (byte index)
    s.insert(5, ',');
    println!("{}", s);  // "hello, world!"
    
    // Insert string at position
    s.insert_str(7, "beautiful ");
    println!("{}", s);  // "hello, beautiful world!"
    
    // Replace
    let new_s = s.replace("beautiful", "wonderful");
    println!("{}", new_s);  // "hello, wonderful world!"
    
    // Truncate (keep first N bytes)
    let mut t = String::from("hello world");
    t.truncate(5);
    println!("{}", t);  // "hello"
    
    // Clear
    t.clear();
    println!("empty: '{}'", t);  // empty: ''
    
    // Remove character at byte index
    let mut r = String::from("hello");
    r.remove(0);
    println!("{}", r);  // "ello"
}
```

### String Growth and Capacity

Like `Vec`, `String` doubles its capacity when it runs out of room:

```rust
fn main() {
    let mut s = String::new();
    
    println!("len: {}, capacity: {}", s.len(), s.capacity());
    // len: 0, capacity: 0
    
    s.push_str("hello");
    println!("len: {}, capacity: {}", s.len(), s.capacity());
    // len: 5, capacity: 8 (typically)
    
    s.push_str(" world");
    println!("len: {}, capacity: {}", s.len(), s.capacity());
    // len: 11, capacity: 16 (typically)
    
    // Pre-allocate to avoid reallocations:
    let mut efficient = String::with_capacity(1000);
    for i in 0..100 {
        efficient.push_str("hello ");
    }
    // Much fewer (possibly zero) reallocations!
}
```

```
Growth pattern:

Push "hello":
  Allocate 8 bytes, copy "hello"
  ┌───────────────────────────┐
  │ h │ e │ l │ l │ o │   │   │   │
  └───────────────────────────┘
    len=5, cap=8

Push " world":
  Need 11 bytes but only have 8 → reallocate to 16
  Free old memory, allocate 16 bytes, copy everything
  ┌───────────────────────────────────────────────────┐
  │ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │   │   │   │   │   │
  └───────────────────────────────────────────────────┘
    len=11, cap=16
```

---

## `&str` — The Borrowed String Slice

A `&str` is a **borrowed, read-only view** into string data. It's a fat pointer: pointer + length.

### What `&str` Looks Like in Memory

```rust
let literal: &str = "hello";         // Points into binary
let s = String::from("hello world");
let slice: &str = &s[0..5];          // Points into heap
```

```
Case 1: String literal
Binary (read-only data section):
┌───┬───┬───┬───┬───┐
│ h │ e │ l │ l │ o │
└───┴───┴───┴───┴───┘
  ↑
Stack:
┌────────────┐
│ ptr ───────┘
│ len: 5     │
└────────────┘
literal: &str (16 bytes)

Case 2: Slice of String
Heap (owned by s):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  ↑
Stack:
┌────────────┐
│ ptr ───────┘
│ len: 5     │
└────────────┘
slice: &str (16 bytes) — points to 'h', length 5
```

### Where `&str` Comes From

```rust
// 1. String literal (embedded in the binary)
let a: &str = "hello";

// 2. Slice of a String
let s = String::from("hello world");
let b: &str = &s[..5];

// 3. Borrowing a whole String (deref coercion)
let c: &str = &s;

// 4. Slice of a byte array (if valid UTF-8)
let bytes = b"hello";
let d: &str = std::str::from_utf8(bytes).unwrap();
```

---

## Side-by-Side Comparison

```
             ┌─────────────────┬──────────────────────┐
             │     String      │       &str            │
             ├─────────────────┼──────────────────────┤
             │ Owned           │ Borrowed              │
             │ Heap-allocated  │ Points anywhere       │
             │ Growable        │ Fixed-size view        │
             │ Mutable         │ Immutable (usually)   │
             │ 24 bytes stack  │ 16 bytes stack        │
             │ + heap data     │ + no heap (borrows)   │
             │ Dropped at end  │ No drop               │
             │   of scope      │   (doesn't own)       │
             │ Like Vec<u8>    │ Like &[u8]            │
             └─────────────────┴──────────────────────┘
```

| Feature | `String` | `&str` |
|---------|----------|--------|
| Ownership | Owns the data | Borrows the data |
| Location | Stack metadata + heap data | Stack (pointer + length) |
| Mutability | Can modify (`push_str`, etc.) | Read-only |
| Size | Variable (grows/shrinks) | Fixed view |
| Allocation | Allocates on heap | No allocation |
| Drop | Yes (frees heap memory) | No (nothing to free) |
| In struct fields | ✅ Easy (owns its data) | ⚠️ Needs lifetime annotations |
| As function param | Works but inflexible | ✅ Preferred |
| Copy | ❌ (would duplicate heap data) | ✅ (just copies ptr + len) |

---

## When to Use Which

### Use `&str` When...

```rust
// ✅ Reading/inspecting string data
fn is_palindrome(s: &str) -> bool {
    let chars: Vec<char> = s.chars().collect();
    let len = chars.len();
    for i in 0..len / 2 {
        if chars[i] != chars[len - 1 - i] {
            return false;
        }
    }
    true
}

// ✅ Function parameters (accept both String and &str)
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

// ✅ Returning a portion of the input
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}
```

### Use `String` When...

```rust
// ✅ Building or constructing new strings
fn build_greeting(name: &str) -> String {
    format!("Hello, {}! Welcome.", name)
}

// ✅ Storing in structs (owned data)
struct User {
    name: String,    // User owns its name
    email: String,
}

// ✅ Modifying strings
fn capitalize(s: &str) -> String {
    let mut result = String::new();
    for (i, c) in s.chars().enumerate() {
        if i == 0 {
            result.extend(c.to_uppercase());
        } else {
            result.push(c);
        }
    }
    result
}

// ✅ Collecting from iterators
fn join_words(words: &[&str]) -> String {
    words.join(", ")
}
```

### Function Parameters: The Golden Rule

**Accept `&str`, not `&String`.**

```rust
// ❌ Unnecessarily restrictive
fn count_vowels(s: &String) -> usize {
    s.chars().filter(|c| "aeiouAEIOU".contains(*c)).count()
}

// ✅ Accepts both String and &str
fn count_vowels(s: &str) -> usize {
    s.chars().filter(|c| "aeiouAEIOU".contains(*c)).count()
}

fn main() {
    let owned = String::from("hello");
    let borrowed = "world";
    
    // With &str parameter, both work:
    count_vowels(&owned);     // &String → &str via deref coercion
    count_vowels(borrowed);   // Already &str
    count_vowels(&owned[1..]); // Slice works too
}
```

Why `&str` is more flexible:

```
&str accepts:
  ├── "literal"           (string literal, already &str)
  ├── &some_string        (borrow of String, auto-deref)
  ├── &some_string[..]    (slice of String)
  └── other_str_ref       (any &str from anywhere)

&String accepts:
  └── &some_string        (ONLY borrow of String)
```

### Function Return Values

```rust
// Return &str when you're returning a portion of the input:
fn first_line(text: &str) -> &str {
    match text.find('\n') {
        Some(i) => &text[..i],
        None => text,
    }
}

// Return String when you're creating new string data:
fn repeat(s: &str, n: usize) -> String {
    s.repeat(n)
}

// Return String when you're computing/combining:
fn full_name(first: &str, last: &str) -> String {
    format!("{} {}", first, last)
}
```

### Struct Fields

```rust
// ✅ Usually: use String for struct fields
struct Config {
    host: String,
    port: u16,
    database: String,
}

// ⚠️ Using &str in structs requires lifetime annotations
// Only do this when borrowing is intentional and the struct
// won't outlive the data it references.
struct SearchResult<'a> {
    line: &'a str,
    line_number: usize,
}
```

Rule of thumb for structs:
- **Use `String`** when the struct should own its data (99% of the time)
- **Use `&str`** only for temporary views where the lifetime relationship is clear

---

## Converting Between String and &str

### String → &str (Cheap)

This is **free** — no allocation, no copying. You're just creating a fat pointer.

```rust
let s = String::from("hello");

// Method 1: as_str()
let slice: &str = s.as_str();

// Method 2: Borrow with &
let slice: &str = &s;

// Method 3: Slice syntax  
let slice: &str = &s[..];

// Method 4: Deref coercion (automatic in many contexts)
fn takes_str(s: &str) {}
takes_str(&s);  // Auto-converts &String to &str
```

```
Cost: O(1) — just creates a pointer, no data copied

String:
┌────────────┐
│ ptr ──────────→ [h][e][l][l][o]   (heap)
│ len: 5     │
│ cap: 5     │
└────────────┘

&str (created from String):
┌────────────┐
│ ptr ──────────→ [h][e][l][l][o]   (same heap data — no copy!)
│ len: 5     │
└────────────┘
```

### &str → String (Allocates)

This **allocates heap memory** and **copies the bytes**. It's not free.

```rust
let slice: &str = "hello";

// Method 1: to_string()
let owned: String = slice.to_string();

// Method 2: String::from()
let owned: String = String::from(slice);

// Method 3: to_owned()
let owned: String = slice.to_owned();

// Method 4: Into trait
let owned: String = slice.into();
```

```
Cost: O(n) — allocates n bytes on heap, copies n bytes

&str:
┌────────────┐
│ ptr ──────────→ [h][e][l][l][o]   (binary/other memory)
│ len: 5     │
└────────────┘

String (created from &str):
┌────────────┐
│ ptr ──────────→ [h][e][l][l][o]   (NEW heap allocation, bytes copied!)
│ len: 5     │
│ cap: 5     │
└────────────┘
```

### Conversion Cost Summary

| Conversion | Cost | Allocates? |
|-----------|------|------------|
| `String` → `&str` | O(1) | No |
| `&str` → `String` | O(n) | Yes |
| `String` → `String` (clone) | O(n) | Yes |
| `&str` → `&str` (copy) | O(1) | No (just copies ptr+len) |

---

## Deref Coercion

Rust automatically converts `&String` to `&str` when needed. This is called **deref coercion**:

```rust
fn print_length(s: &str) {
    println!("Length: {}", s.len());
}

fn main() {
    let owned = String::from("hello");
    
    print_length(&owned);  // &String automatically becomes &str
    //          ^^^^^^
    //          Rust sees &String but function wants &str
    //          It auto-calls: &*owned → &str
}
```

How it works under the hood:

```
&String
   ↓  Deref: String implements Deref<Target = str>
   ↓  So &String → &str automatically
&str
```

This also works in chains:

```rust
let boxed: Box<String> = Box::new(String::from("hello"));
print_length(&boxed);  // &Box<String> → &String → &str
```

---

## String Internals: UTF-8

Both `String` and `&str` are **always valid UTF-8**. This has important consequences.

### Indexing Strings

You **cannot** index a string by character position:

```rust
let s = String::from("hello");
// let c = s[0];  // ❌ ERROR: String cannot be indexed by integer

// Why? Because characters can be multiple bytes:
let russian = String::from("Здравствуйте");
// russian[0] would be... byte 0? That's only half of 'З'!
```

The compiler error:

```
error[E0277]: the type `str` cannot be indexed by `{integer}`
```

**Instead, use these approaches:**

```rust
let s = "hello";

// Get bytes (if you know it's ASCII)
let first_byte = s.as_bytes()[0];  // 104 (ASCII for 'h')

// Get characters
let first_char = s.chars().nth(0);  // Some('h')

// Get a substring (byte range)
let sub = &s[0..1];  // "h" — works for ASCII
```

### Iterating Over Characters

```rust
fn main() {
    let s = "Hello 🌍";
    
    // By character (what you usually want):
    for c in s.chars() {
        print!("'{}' ", c);
    }
    println!();
    // 'H' 'e' 'l' 'l' 'o' ' ' '🌍'
    
    // By byte (low level):
    for b in s.bytes() {
        print!("{} ", b);
    }
    println!();
    // 72 101 108 108 111 32 240 159 140 141
    
    // By character with byte index:
    for (i, c) in s.char_indices() {
        println!("byte {}: '{}'", i, c);
    }
    // byte 0: 'H'
    // byte 1: 'e'
    // byte 2: 'l'
    // byte 3: 'l'
    // byte 4: 'o'
    // byte 5: ' '
    // byte 6: '🌍'  (4 bytes: 6, 7, 8, 9)
}
```

### Getting the Nth Character

```rust
fn nth_char(s: &str, n: usize) -> Option<char> {
    s.chars().nth(n)
}

fn main() {
    let s = "Hello 🌍";
    println!("{:?}", nth_char(s, 0));  // Some('H')
    println!("{:?}", nth_char(s, 6));  // Some('🌍')
    println!("{:?}", nth_char(s, 7));  // None
}
```

Note: `chars().nth(n)` is O(n) because it has to walk through the bytes to count characters. If you need random access by character index, convert to `Vec<char>` first.

---

## Common String Operations

```rust
fn main() {
    // Concatenation
    let hello = String::from("Hello");
    let world = String::from("World");
    
    // Using + operator (takes ownership of left side)
    let combined = hello + " " + &world;
    // hello is moved! Can't use it anymore.
    // world is borrowed, still usable.
    println!("{}", combined);  // "Hello World"
    
    // Using format! (doesn't take ownership of anything)
    let a = String::from("Hello");
    let b = String::from("World");
    let c = format!("{} {}", a, b);
    println!("{} {} {}", a, b, c);  // All still usable!
    
    // Comparison
    let s1 = String::from("hello");
    let s2 = "hello";
    println!("{}", s1 == s2);  // true (String == &str works!)
    
    // Case conversion (returns new String)
    let upper = "hello".to_uppercase();
    let lower = "HELLO".to_lowercase();
    println!("{} {}", upper, lower);  // "HELLO hello"
    
    // Trimming (returns &str)
    let padded = "  hello  ";
    let trimmed: &str = padded.trim();
    println!("'{}'", trimmed);  // 'hello'
    
    // Splitting (returns iterator of &str)
    let csv = "a,b,c,d";
    let parts: Vec<&str> = csv.split(',').collect();
    println!("{:?}", parts);  // ["a", "b", "c", "d"]
    
    // Contains, starts_with, ends_with
    let s = "Hello, World!";
    println!("{}", s.contains("World"));      // true
    println!("{}", s.starts_with("Hello"));   // true
    println!("{}", s.ends_with("!"));         // true
    
    // Replacing
    let new = "Hello, World!".replace("World", "Rust");
    println!("{}", new);  // "Hello, Rust!"
}
```

### The `+` Operator Trap

The `+` operator has a surprising signature:

```rust
// Simplified: fn add(self, s: &str) -> String
//             ^^^^        ^^^^
//             Takes        Borrows
//             ownership    right side
//             of left

let s1 = String::from("Hello ");
let s2 = String::from("World");

let s3 = s1 + &s2;  // s1 is moved into s3
// println!("{}", s1);  // ❌ s1 was moved
println!("{}", s2);     // ✅ s2 was only borrowed
println!("{}", s3);     // "Hello World"
```

When concatenating many strings, prefer `format!`:

```rust
let first = "John";
let middle = "Michael";
let last = "Smith";

// ❌ Awkward with +
let full = first.to_string() + " " + middle + " " + last;

// ✅ Clean with format!
let full = format!("{} {} {}", first, middle, last);
```

---

## `to_string()` vs `to_owned()` vs `String::from()`

All three convert `&str` to `String`. They differ subtly:

```rust
let s1 = "hello".to_string();    // Uses Display trait
let s2 = "hello".to_owned();     // Uses ToOwned trait
let s3 = String::from("hello");  // Uses From trait

// Result is identical for &str. Performance is the same.
```

| Method | Trait | Works On | Idiomatic For |
|--------|-------|----------|---------------|
| `.to_string()` | `Display` | Anything with Display | Converting non-strings (`42.to_string()`) |
| `.to_owned()` | `ToOwned` | Borrowed types | Converting `&str` → `String`, `&[T]` → `Vec<T>` |
| `String::from()` | `From` | Types implementing From | Direct construction from `&str` |

In practice, all three are used interchangeably for `&str` → `String`. Pick one and be consistent. `String::from()` and `.to_owned()` are slightly more idiomatic for this specific case.

---

## `&String` vs `&str`

You might wonder: why not just use `&String` everywhere?

```rust
fn takes_ref_string(s: &String) { /* ... */ }   // Works but limited
fn takes_str(s: &str) { /* ... */ }              // Better!
```

`&String` and `&str` are different types:

```
&String (reference to a String struct):
┌──────────┐
│ ptr ─────────→ ┌──────────┐
└──────────┘     │ ptr ─────────→ [h][e][l][l][o]  (heap)
  8 bytes        │ len: 5   │
                 │ cap: 5   │
                 └──────────┘
  Points to a String struct (which then points to heap data)
  Two levels of indirection!

&str (fat pointer directly to data):
┌──────────┐
│ ptr ─────────→ [h][e][l][l][o]  (anywhere: heap, binary, stack)
│ len: 5   │
└──────────┘
  16 bytes
  Directly points to the data — one level of indirection
```

`&str` is:
1. **More flexible** — accepts `&String`, `&str`, and slices
2. **More efficient** — one fewer level of indirection
3. **Idiomatic** — the Rust community convention

---

## Strings in Collections

```rust
use std::collections::HashMap;

fn main() {
    // Vec of owned strings
    let mut names: Vec<String> = Vec::new();
    names.push(String::from("Alice"));
    names.push("Bob".to_string());
    
    // Vec of borrowed strings (needs lifetime)
    let borrowed: Vec<&str> = vec!["Alice", "Bob", "Charlie"];
    
    // HashMap with String keys
    let mut scores: HashMap<String, i32> = HashMap::new();
    scores.insert(String::from("Alice"), 100);
    scores.insert("Bob".to_string(), 85);
    
    // Lookup with &str (no need to create String for lookup!)
    if let Some(score) = scores.get("Alice") {
        println!("Alice: {}", score);
    }
    
    // This works because HashMap::get accepts &Q where K: Borrow<Q>
    // String implements Borrow<str>, so you can look up with &str
}
```

---

## Real-World Patterns

### Pattern 1: Parsing Configuration

```rust
struct Config {
    host: String,
    port: u16,
}

fn parse_config(input: &str) -> Config {
    let mut host = String::from("localhost");
    let mut port = 8080;
    
    for line in input.lines() {
        if let Some(value) = line.strip_prefix("host=") {
            host = value.trim().to_string();
        } else if let Some(value) = line.strip_prefix("port=") {
            port = value.trim().parse().unwrap_or(8080);
        }
    }
    
    Config { host, port }
}
```

### Pattern 2: Building Output

```rust
fn generate_report(items: &[(&str, f64)]) -> String {
    let mut report = String::with_capacity(256);
    
    report.push_str("=== Report ===\n");
    
    let mut total = 0.0;
    for (name, price) in items {
        report.push_str(&format!("  {}: ${:.2}\n", name, price));
        total += price;
    }
    
    report.push_str(&format!("  Total: ${:.2}\n", total));
    report.push_str("==============\n");
    
    report
}

fn main() {
    let items = vec![
        ("Widget", 9.99),
        ("Gadget", 24.95),
        ("Doohickey", 4.50),
    ];
    
    let report = generate_report(&items);
    println!("{}", report);
}
```

### Pattern 3: Accepting Any String-Like Type

```rust
// Accept anything that can be converted to &str:
fn log_message(msg: impl AsRef<str>) {
    let s: &str = msg.as_ref();
    println!("[LOG] {}", s);
}

fn main() {
    log_message("hello");                    // &str
    log_message(String::from("world"));      // String
    log_message(&String::from("owned ref")); // &String
}
```

### Pattern 4: Cow — Clone on Write

For cases where you sometimes need to own and sometimes borrow:

```rust
use std::borrow::Cow;

fn process_name(name: &str) -> Cow<str> {
    if name.contains(' ') {
        // Need to create a new string
        Cow::Owned(name.replace(' ', "_"))
    } else {
        // Can just borrow the original
        Cow::Borrowed(name)
    }
}

fn main() {
    let name1 = process_name("Alice");        // Cow::Borrowed — no allocation
    let name2 = process_name("Bob Smith");    // Cow::Owned — allocated "Bob_Smith"
    
    println!("{}", name1);  // "Alice"
    println!("{}", name2);  // "Bob_Smith"
}
```

---

## Common Mistakes

### Mistake 1: Using `&String` in Function Parameters

```rust
// ❌ Don't do this
fn process(s: &String) {}

// ✅ Do this
fn process(s: &str) {}
```

### Mistake 2: Cloning When Borrowing Would Work

```rust
fn greet(name: &str) {
    // ❌ Unnecessary clone
    let owned = name.to_string();
    println!("Hello, {}!", owned);
    
    // ✅ Just use the &str directly
    println!("Hello, {}!", name);
}
```

### Mistake 3: Indexing By Character Position

```rust
let s = "Hello 🌍";

// ❌ Can't index by integer
// let c = s[6];

// ✅ Use chars()
let c = s.chars().nth(6);  // Some('🌍')

// ❌ Byte indexing might panic with multi-byte chars
// let sub = &s[6..7];  // PANIC: byte 7 is mid-character

// ✅ Use char_indices for safe slicing
```

### Mistake 4: Forgetting + Takes Ownership

```rust
let a = String::from("hello");
let b = String::from(" world");

let c = a + &b;
// println!("{}", a);  // ❌ a was moved!
println!("{}", b);     // ✅ b was only borrowed
```

---

## Exercises

### Exercise 1: Count Words

Write a function that counts words in a string (words are separated by whitespace):

```rust
fn count_words(s: &str) -> usize {
    // Your code here
}

fn main() {
    assert_eq!(count_words("hello world"), 2);
    assert_eq!(count_words("  spaces   between  "), 2);
    assert_eq!(count_words(""), 0);
    assert_eq!(count_words("one"), 1);
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn count_words(s: &str) -> usize {
    s.split_whitespace().count()
}
```

</details>

### Exercise 2: Reverse Words

Write a function that reverses the order of words in a string:

```rust
fn reverse_words(s: &str) -> String {
    // Your code here
}

fn main() {
    assert_eq!(reverse_words("hello world"), "world hello");
    assert_eq!(reverse_words("one two three"), "three two one");
    assert_eq!(reverse_words("single"), "single");
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn reverse_words(s: &str) -> String {
    s.split_whitespace()
        .rev()
        .collect::<Vec<&str>>()
        .join(" ")
}
```

</details>

### Exercise 3: Type Annotations

What is the type of each variable?

```rust
fn main() {
    let a = "hello";                    // ?
    let b = String::from("hello");      // ?
    let c = &b;                         // ?
    let d = b.as_str();                 // ?
    let e = &b[0..3];                   // ?
    let f = b.clone();                  // ?
    let g: &str = &b;                   // ?
}
```

<details>
<summary>Answers</summary>

```rust
let a = "hello";                    // &str (string literal)
let b = String::from("hello");      // String
let c = &b;                         // &String
let d = b.as_str();                 // &str
let e = &b[0..3];                   // &str
let f = b.clone();                  // String
let g: &str = &b;                   // &str (deref coercion from &String)
```

</details>

### Exercise 4: Title Case

Write a function that converts a string to title case (first letter of each word capitalized):

```rust
fn to_title_case(s: &str) -> String {
    // Your code here
}

fn main() {
    assert_eq!(to_title_case("hello world"), "Hello World");
    assert_eq!(to_title_case("rust is great"), "Rust Is Great");
    println!("All tests passed!");
}
```

<details>
<summary>Solution</summary>

```rust
fn to_title_case(s: &str) -> String {
    s.split_whitespace()
        .map(|word| {
            let mut chars = word.chars();
            match chars.next() {
                None => String::new(),
                Some(first) => {
                    let upper: String = first.to_uppercase().collect();
                    upper + chars.as_str()
                }
            }
        })
        .collect::<Vec<String>>()
        .join(" ")
}
```

</details>

---

## Summary

### Quick Reference

```
String                          &str
──────                          ────
Owned                           Borrowed
Heap-allocated                  Points to any string data
Growable, mutable               Fixed-size, read-only
24 bytes on stack + heap        16 bytes on stack
Uses: storing, building         Uses: reading, passing
Drop: frees heap                Drop: nothing (doesn't own)

Conversions:
  String → &str    FREE      .as_str(), &s, deref coercion
  &str → String    ALLOCATES .to_string(), .to_owned(), String::from()
```

### Decision Flowchart

```
Do you need to own the string data?
├── YES → Use String
│   ├── Storing in a struct? → String
│   ├── Building a new string? → String
│   └── Returning computed text? → String
└── NO → Use &str
    ├── Function parameter? → &str
    ├── Just reading? → &str
    └── Returning a piece of input? → &str
```

### Key Takeaways

1. **`String` = owned, heap, growable** — like `Vec<u8>` with UTF-8 guarantee
2. **`&str` = borrowed, fat pointer (ptr + len)** — a view into string data
3. **Prefer `&str` in function parameters** — more flexible due to deref coercion
4. **Use `String` in struct fields** — structs usually own their data
5. **`String` → `&str` is free (O(1))**, `&str` → `String` allocates (O(n))
6. **Strings are UTF-8** — you can't index by character position
7. **Use `format!` for combining strings** — cleaner than `+` operator
8. **`to_string()`, `to_owned()`, `String::from()` all do the same thing for `&str`**

---

## What's Next?

Congratulations! You've completed Stage 3: Ownership! You now understand:
- Stack and heap memory
- Ownership rules and moves
- Clone and Copy traits
- References and borrowing
- Mutable references
- Slices
- String vs &str

These concepts are **foundational to everything in Rust**. Every future topic builds on ownership. In the next stage, we'll use ownership knowledge to **structure data** with structs and enums.

**Next Stage:** [Stage 4: Structuring Data →](../04-structuring-data/README.md)

---

<p align="center">
  <i>Tutorial 8 of 8 — Stage 3: Ownership</i>
</p>

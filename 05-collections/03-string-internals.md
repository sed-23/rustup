# String Internals 🔤

> **Rust's `String` is a `Vec<u8>` that guarantees valid UTF-8. Understanding UTF-8 encoding, the difference between bytes, chars, and graphemes, and how string operations work is essential for effective Rust programming.**

---

## Table of Contents

- [String vs &str](#string-vs-str)
- [String Memory Layout](#string-memory-layout)
- [UTF-8 Encoding](#utf-8-encoding)
- [Bytes, Chars, and Graphemes](#bytes-chars-and-graphemes)
- [Creating Strings](#creating-strings)
- [String Operations](#string-operations)
- [Indexing Strings](#indexing-strings)
- [Slicing Strings](#slicing-strings)
- [String Formatting](#string-formatting)
- [Conversion](#conversion)
- [Common Patterns](#common-patterns)
- [Exercises](#exercises)
- [Summary](#summary)

---

## String vs &str

| Feature | `String` | `&str` |
|---------|----------|--------|
| Ownership | Owned | Borrowed |
| Location | Heap | Anywhere (heap, stack, binary) |
| Mutable | Yes (`mut`) | No |
| Growable | Yes | No |
| Internal | `Vec<u8>` | `&[u8]` (fat pointer) |
| Size on stack | 24 bytes | 16 bytes (ptr + len) |

```rust
fn main() {
    let s: String = String::from("hello");  // Owned, on heap
    let r: &str = &s;                        // Borrow of String
    let lit: &str = "world";                 // String literal (in binary)
}
```

---

## String Memory Layout

```
String (on stack, 24 bytes):          Heap:
┌────────────────────┐               ┌───┬───┬───┬───┬───┐
│ ptr ──────────────────────────→     │ h │ e │ l │ l │ o │  (UTF-8 bytes)
│ len: 5             │               └───┴───┴───┴───┴───┘
│ cap: 5             │
└────────────────────┘

&str (on stack, 16 bytes):
┌────────────────────┐
│ ptr ──────────────────→ (somewhere in memory)
│ len: 5             │
└────────────────────┘
```

---

## UTF-8 Encoding

Rust stores all strings as UTF-8. Here's how UTF-8 encodes Unicode:

```
Code point range       | Bytes | Bit pattern
U+0000   – U+007F     | 1     | 0xxxxxxx
U+0080   – U+07FF     | 2     | 110xxxxx 10xxxxxx
U+0800   – U+FFFF     | 3     | 1110xxxx 10xxxxxx 10xxxxxx
U+10000  – U+10FFFF   | 4     | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

```rust
fn main() {
    // ASCII: 1 byte each
    let ascii = "hello";
    println!("bytes: {}", ascii.len());  // 5
    
    // German: ä = 2 bytes
    let german = "Ärger";
    println!("bytes: {}, chars: {}", german.len(), german.chars().count());
    // bytes: 6, chars: 5  (Ä = 2 bytes)
    
    // Chinese: each char = 3 bytes
    let chinese = "你好";
    println!("bytes: {}, chars: {}", chinese.len(), chinese.chars().count());
    // bytes: 6, chars: 2
    
    // Emoji: 4 bytes
    let emoji = "😀";
    println!("bytes: {}, chars: {}", emoji.len(), emoji.chars().count());
    // bytes: 4, chars: 1
}
```

---

### The History of Text Encoding — From Telegraph to UTF-8

UTF-8 didn't appear out of nowhere. It's the survivor of a 150-year evolution in how
humans encode text into machines. Understanding this history explains *why* Rust made
UTF-8 its default and why so many other encodings still haunt us.

**The timeline:**

- **Baudot code (1870)** — The first practical character encoding, designed for telegraphs.
  Only 5 bits → 32 possible characters. Letters were uppercase only, and you toggled a
  "shift" to access numbers. Used well into the 1960s on telex machines.

- **ASCII (1963)** — 7-bit encoding → 128 characters. Covered uppercase, lowercase, digits,
  punctuation, and 33 control codes (like newline and bell). Designed for American English.
  Became *the* standard for decades — every encoding that came after had to deal with ASCII
  compatibility.

- **The chaos of the 1980s** — Every region on Earth invented their own extension of ASCII:
  - **Latin-1 (ISO 8859-1)** — Western European: é, ñ, ü
  - **Shift-JIS** — Japanese: mixed single/double-byte encoding
  - **GB2312** — Simplified Chinese
  - **KOI8-R** — Russian Cyrillic
  - **Windows-1252** — Microsoft's "Latin-1 but different"

  None of these were compatible with each other.

- **"Mojibake" (文字化け)** — Japanese term for garbled text that appears when you open a
  file with the wrong encoding. Example: "café" in Latin-1 opened as UTF-8 → "cafÃ©".
  This *still* happens today in email attachments, CSV exports, and legacy databases.

- **Unicode (1991)** — The Unicode Consortium set out to create ONE encoding for ALL writing
  systems. Version 1.0 had ~7,000 characters. Today (Unicode 15.1): **149,813 characters**
  covering 161 scripts, including Egyptian hieroglyphs and emoji.

- **UTF-16 (1996)** — Early Unicode bet that 65,536 code points would be enough, so they
  used fixed 2-byte units. They were wrong. To handle code points above U+FFFF, they added
  "surrogate pairs" — a messy hack where one character takes 4 bytes. Java and JavaScript
  strings are UTF-16, which is why `"😀".length === 2` in JavaScript.

- **UTF-8 (1992)** — Ken Thompson and Rob Pike designed it *on a placemat at a diner* in
  New Jersey. The key insight: use variable-length encoding (1-4 bytes) where ASCII
  characters stay exactly the same — one byte, same value. This meant every existing ASCII
  file was already valid UTF-8.

**Why UTF-8 won:**

```
✅ Backward-compatible with ASCII (every ASCII file is valid UTF-8)
✅ No byte-order issues (no BOM needed, unlike UTF-16)
✅ Self-synchronizing (you can jump into the middle and find char boundaries)
✅ Space-efficient for Latin text (1 byte/char vs UTF-16's 2 bytes/char)
✅ No surrogate pair hacks
```

| Encoding   | Year | Bytes/Char  | Max Characters | Still Used?              |
|------------|------|-------------|----------------|--------------------------|
| Baudot     | 1870 | 0.625 (5-bit)| 32            | No (museums only)        |
| ASCII      | 1963 | 1           | 128            | Yes (subset of UTF-8)    |
| Latin-1    | 1985 | 1           | 256            | Legacy databases, HTTP   |
| Shift-JIS  | 1982 | 1-2         | ~7,000         | Legacy Japanese software |
| UTF-16     | 1996 | 2-4         | 1,112,064      | Java, JavaScript, Win32  |
| UTF-8      | 1992 | 1-4         | 1,112,064      | **98%+ of all websites** |

Today, UTF-8 is the default encoding for Rust, Go, Python 3, and most modern tools.
Rust's decision to make `String` always UTF-8 is not arbitrary — it's following the
clear winner of 150 years of encoding history.

---

## Bytes, Chars, and Graphemes

A "character" on screen can be multiple Unicode code points:

```rust
fn main() {
    // The flag emoji 🇺🇸 is TWO code points: U+1F1FA U+1F1F8
    let flag = "🇺🇸";
    println!("bytes: {}", flag.len());             // 8
    println!("chars: {}", flag.chars().count());   // 2
    // But visually it's 1 "grapheme cluster"
    
    // é can be 1 char (U+00E9) or 2 chars (e + ´ combining accent)
    let e1 = "é";     // precomposed: 1 char, 2 bytes
    let e2 = "é";     // decomposed: 2 chars (e + combining acute)
    
    // Three views of a string:
    let s = "नमस्ते";
    
    // As bytes
    println!("Bytes: {:?}", s.as_bytes());
    // [224, 164, 168, 224, 164, 174, 224, 164, 184, 224, 165, 141, 224, 164, 164, 224, 165, 135]
    
    // As chars (Unicode scalar values)
    println!("Chars: {:?}", s.chars().collect::<Vec<_>>());
    // ['न', 'म', 'स', '्', 'त', 'े']  — 6 chars
    
    // As grapheme clusters (what humans see) — requires `unicode-segmentation` crate
    // "न", "म", "स्", "ते" — 4 graphemes
}
```

---

### Bytes, Chars, and Graphemes — The Unicode Iceberg

The three "levels" of looking at a string are like an iceberg — what you see on screen
(graphemes) hides enormous complexity underneath (chars and bytes).

```
    What you see:     "é"        (1 grapheme cluster)
                       │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
    Level 1:       "é" (U+00E9)      OR  "e" + "◌́" (U+0065 + U+0301)
    (chars)         1 scalar              2 scalars
    │                  │                  │
    Level 2:       [0xC3, 0xA9]      OR  [0x65, 0xCC, 0x81]
    (bytes)          2 bytes               3 bytes
```

**Canonical equivalence:** The precomposed `é` (U+00E9) and the decomposed `e` + combining
acute accent (U+0065 + U+0301) look *identical* on screen but have completely different
byte sequences. If you compare them byte-by-byte, they won't match!

**Unicode normalization** solves this:
- **NFC** (Canonical Decomposition, then Canonical Composition) — "compose" into fewest
  code points: `e + ◌́ → é`
- **NFD** (Canonical Decomposition) — "decompose" into base + combining marks: `é → e + ◌́`

You **must** normalize strings before comparing or hashing them if they come from
different sources. Rust's standard library doesn't include normalization — use the
`unicode-normalization` crate.

**Grapheme clusters — what a human sees as "a character":**

```
Example              Code Points               Scalars  Bytes  Graphemes
─────────────────────────────────────────────────────────────────────────
"e"                  U+0065                        1      1       1
"é"                  U+00E9                        1      2       1
"é"                  U+0065 U+0301                 2      3       1
"🇺🇸"                 U+1F1FA U+1F1F8               2      8       1
"👨‍👩‍👧‍👦"              U+1F468 U+200D U+1F469        7     25       1
                     U+200D U+1F467 U+200D
                     U+1F466
"நி"                  U+0BA8 U+0BBF                 2      6       1
```

**The flag emoji deep-dive:** 🇺🇸 is not a single Unicode code point. It's TWO "Regional
Indicator" symbols: 🇺 (U+1F1FA) + 🇸 (U+1F1F8). The rendering engine sees two regional
indicators and combines them into a flag. Rust's `char` type can hold each indicator
separately, but there is no single `char` for "the US flag."

**The family emoji deep-dive:** 👨‍👩‍👧‍👦 is SEVEN Unicode scalars joined by Zero-Width Joiners:
`👨 + ZWJ + 👩 + ZWJ + 👧 + ZWJ + 👦`. That's 25 bytes of UTF-8 for one visible glyph.

**Why Rust's `char` is NOT a "character":**
A Rust `char` is a Unicode Scalar Value — a 21-bit code point (stored in 4 bytes for
alignment). It can represent individual scalars like `'a'` or `'🇺'`, but it fundamentally
*cannot* represent a grapheme cluster like 🇺🇸 or 👨‍👩‍👧‍👦 as a single value.

**Why isn't grapheme segmentation in `std`?**
Grapheme cluster rules are defined by Unicode Annex #29 and change with *every Unicode
version*. Baking this into `std` would tie Rust's release cycle to Unicode's. Instead,
the ecosystem crate `unicode-segmentation` handles this:

```rust
// In Cargo.toml: unicode-segmentation = "1.10"
use unicode_segmentation::UnicodeSegmentation;

fn main() {
    let family = "👨\u{200D}👩\u{200D}👧\u{200D}👦";
    let graphemes: Vec<&str> = family.graphemes(true).collect();
    println!("{:?}", graphemes);  // ["👨\u{200d}👩\u{200d}👧\u{200d}👦"]
    println!("grapheme count: {}", graphemes.len());  // 1
}
```

---

## Creating Strings

```rust
fn main() {
    // From literal
    let s = String::from("hello");
    let s = "hello".to_string();
    let s = "hello".to_owned();
    
    // Empty
    let s = String::new();
    
    // With capacity
    let s = String::with_capacity(100);
    
    // From chars
    let s: String = ['h', 'i'].iter().collect();
    
    // From bytes (fallible — must be valid UTF-8)
    let bytes = vec![104, 101, 108, 108, 111];
    let s = String::from_utf8(bytes).unwrap();  // "hello"
    
    // From bytes (lossy — replaces invalid with ?)
    let bad = vec![104, 101, 0xFF, 108, 111];
    let s = String::from_utf8_lossy(&bad);  // "he�lo"
    
    // Repeat
    let s = "ha".repeat(3);  // "hahaha"
}
```

---

## String Operations

```rust
fn main() {
    let mut s = String::from("hello");
    
    // Append
    s.push(' ');              // Push a char
    s.push_str("world");     // Push a &str
    println!("{}", s);        // "hello world"
    
    // Concatenation with +
    let s1 = String::from("hello");
    let s2 = String::from(" world");
    let s3 = s1 + &s2;       // s1 is MOVED, s2 is borrowed
    // s1 can't be used anymore
    
    // format! macro (doesn't move anything)
    let s1 = String::from("hello");
    let s2 = String::from("world");
    let s3 = format!("{} {}", s1, s2);  // Both still usable
    
    // Replace
    let s = "I like cats".replace("cats", "dogs");
    // "I like dogs"
    
    // Replace first n occurrences
    let s = "aaa".replacen("a", "b", 2);  // "bba"
    
    // Trim
    let s = "  hello  ".trim();          // "hello"
    let s = "  hello  ".trim_start();    // "hello  "
    let s = "  hello  ".trim_end();      // "  hello"
    
    // Case
    let upper = "hello".to_uppercase();   // "HELLO"
    let lower = "HELLO".to_lowercase();   // "hello"
    
    // Contains / starts / ends
    let s = "hello world";
    println!("{}", s.contains("world"));     // true
    println!("{}", s.starts_with("hello"));  // true
    println!("{}", s.ends_with("ld"));       // true
    
    // Split
    let parts: Vec<&str> = "a,b,c".split(',').collect();
    // ["a", "b", "c"]
    
    let words: Vec<&str> = "hello   world".split_whitespace().collect();
    // ["hello", "world"]
    
    // Lines
    let lines: Vec<&str> = "line1\nline2\nline3".lines().collect();
}
```

---

## Indexing Strings

**You cannot index a `String` with a single number:**

```rust
fn main() {
    let s = String::from("hello");
    // let c = s[0];  // ❌ ERROR: String cannot be indexed by integer
}
```

Why? Because UTF-8 characters have variable length:

```
"hello" → [104, 101, 108, 108, 111]  bytes
            h    e    l    l    o     chars
Index:      0    1    2    3    4     ← byte offsets match char index

"Здравствуйте" → each Cyrillic letter = 2 bytes
Byte index 0 is just half of 'З' — meaningless alone
```

---

## Slicing Strings

You can slice by **byte range**, but it must fall on character boundaries:

```rust
fn main() {
    let s = "Здравствуйте";
    
    // First 4 BYTES = first 2 Cyrillic chars
    let slice = &s[0..4];  // "Зд"
    
    // This panics — byte 1 is in the middle of 'З'
    // let bad = &s[0..1];  // PANIC: not a char boundary
    
    // Safe alternative: use chars
    let first_char: char = s.chars().nth(0).unwrap();  // 'З'
    
    // Get first n characters as &str
    let first_3: String = s.chars().take(3).collect();  // "Здр"
}
```

---

## String Formatting

```rust
fn main() {
    // Padding
    println!("{:>10}", "right");   // "     right"
    println!("{:<10}", "left");    // "left      "
    println!("{:^10}", "center");  // "  center  "
    println!("{:*^10}", "hi");     // "****hi****"
    
    // Numbers
    println!("{:05}", 42);         // "00042"
    println!("{:+}", 42);          // "+42"
    println!("{:#b}", 42);         // "0b101010"
    println!("{:#o}", 42);         // "0o52"
    println!("{:#x}", 42);         // "0x2a"
    println!("{:.2}", 3.14159);    // "3.14"
    
    // Debug vs Display
    println!("{:?}", "hello");     // "\"hello\""
    println!("{}", "hello");       // "hello"
}
```

---

## Conversion

```rust
fn main() {
    // String ↔ &str
    let s: String = String::from("hello");
    let r: &str = &s;             // String → &str (free, just borrows)
    let s: String = r.to_string(); // &str → String (heap allocation)
    
    // Number → String
    let s = 42.to_string();       // "42"
    let s = format!("{:.2}", 3.14); // "3.14"
    
    // String → Number
    let n: i32 = "42".parse().unwrap();
    let f: f64 = "3.14".parse().unwrap();
    
    // String ↔ bytes
    let bytes: &[u8] = s.as_bytes();
    let s = String::from_utf8(vec![104, 105]).unwrap();
    
    // String ↔ chars
    let chars: Vec<char> = "hello".chars().collect();
    let s: String = chars.into_iter().collect();
    
    // &str ↔ String in function args
    fn greet(name: &str) {  // Accepts both String and &str
        println!("Hello, {}!", name);
    }
    greet("world");                      // &str
    greet(&String::from("world"));       // &String → &str via Deref
}
```

---

## Common Patterns

```rust
fn main() {
    // Build string piece by piece
    let mut result = String::new();
    for i in 0..5 {
        if !result.is_empty() {
            result.push_str(", ");
        }
        result.push_str(&i.to_string());
    }
    println!("{}", result);  // "0, 1, 2, 3, 4"
    
    // Join (cleaner)
    let s: String = (0..5)
        .map(|i| i.to_string())
        .collect::<Vec<_>>()
        .join(", ");
    println!("{}", s);  // "0, 1, 2, 3, 4"
    
    // Check if ASCII
    println!("{}", "hello".is_ascii());  // true
    println!("{}", "héllo".is_ascii());  // false
    
    // Cow<str> — avoid allocation when not needed
    use std::borrow::Cow;
    fn maybe_uppercase(s: &str) -> Cow<str> {
        if s.chars().any(|c| c.is_lowercase()) {
            Cow::Owned(s.to_uppercase())
        } else {
            Cow::Borrowed(s)  // No allocation
        }
    }
}
```

---

### String Performance — Allocation Patterns and Optimization

Under the hood, `String` is a `Vec<u8>` with a UTF-8 invariant. This means *all*
`Vec` optimization advice applies directly to `String`.

**Pre-allocate with `String::with_capacity(n)`:**

```rust
// BAD: multiple reallocations as the string grows
let mut s = String::new();
for word in &words {
    s.push_str(word);  // May reallocate on each push
}

// GOOD: one allocation up front
let total_len: usize = words.iter().map(|w| w.len()).sum();
let mut s = String::with_capacity(total_len);
for word in &words {
    s.push_str(word);  // No reallocations
}
```

**The `+` operator trap — O(n²) in a loop:**

Each `+` creates a new `String` allocation. In a loop of n iterations, this means
n allocations and O(n²) total bytes copied:

```rust
// ❌ O(n²) — each + allocates a NEW String
let mut result = String::new();
for item in &items {
    result = result + item;  // Moves old result, allocates new
}

// ✅ O(n) — push_str appends in place
let mut result = String::with_capacity(estimated_size);
for item in &items {
    result.push_str(item);  // Appends to existing buffer
}
```

**`format!()` vs `write!()` — allocation control:**

```rust
use std::fmt::Write;

// format! always allocates a NEW String
let s = format!("Hello, {}!", name);  // New allocation

// write! appends to an EXISTING String — no new allocation
let mut s = String::with_capacity(256);
write!(s, "Hello, {}!", name).unwrap();   // Writes into s
write!(s, " You have {} items.", count).unwrap();  // Still the same buffer
```

**Real-world example — building a URL with query parameters:**

```rust
fn build_url(base: &str, params: &[(&str, &str)]) -> String {
    // Estimate: base + "?" + params as "key=value&" (~20 bytes each)
    let mut url = String::with_capacity(base.len() + params.len() * 20);
    url.push_str(base);
    for (i, (key, value)) in params.iter().enumerate() {
        url.push(if i == 0 { '?' } else { '&' });
        url.push_str(key);
        url.push('=');
        url.push_str(value);
    }
    url  // One allocation, no reallocations for typical URLs
}
```

**The `collect` trick — efficient because of `size_hint`:**

```rust
let s: String = some_chars.collect();  // Efficient!
// Rust calls size_hint() on the iterator to pre-allocate the right capacity.
// For example, chars from a &str know their byte length upper bound.
```

**`Cow<str>` — borrow when you can, allocate when you must:**

```rust
use std::borrow::Cow;

fn clean_input(input: &str) -> Cow<str> {
    if input.contains('\t') {
        Cow::Owned(input.replace('\t', "    "))  // Must allocate
    } else {
        Cow::Borrowed(input)  // Zero-cost: just returns the reference
    }
}
// In practice, most inputs won't have tabs → most calls are free.
```

**When `String` is overkill — cheaper alternatives for immutable text:**

```
Type        Stack Size   Heap         Growable   When to Use
─────────── ────────── ──────────── ────────── ──────────────────────────────
&str          16 B     No (borrow)    No       Function params, temp views
String        24 B     Yes (owned)    Yes      Building/modifying text  
Box<str>      16 B     Yes (owned)    No       Owned immutable text (saves 8B)
Arc<str>      16 B     Yes (shared)   No       Shared across threads
Cow<str>      24 B     Maybe          No       Sometimes modify, usually don't
```

If you never modify a string after creation, `Box<str>` saves 8 bytes on the stack
(no `capacity` field) and the exact heap size (no excess capacity). For shared
immutable strings across threads, `Arc<str>` avoids cloning entirely.

---

## Exercises

### Exercise 1: Character Count

Write a function that returns a `HashMap<char, usize>` counting each character:

```rust
fn char_count(s: &str) -> HashMap<char, usize> {
    todo!()
}
```

<details>
<summary>Solution</summary>

```rust
use std::collections::HashMap;

fn char_count(s: &str) -> HashMap<char, usize> {
    let mut counts = HashMap::new();
    for c in s.chars() {
        *counts.entry(c).or_insert(0) += 1;
    }
    counts
}
```

</details>

### Exercise 2: Pig Latin

Convert a word to Pig Latin: move first consonant to end + "ay", or add "hay" if starts with vowel:

```rust
fn pig_latin(word: &str) -> String {
    todo!()
}
// pig_latin("first") → "irstfay"
// pig_latin("apple") → "applehay"
```

<details>
<summary>Solution</summary>

```rust
fn pig_latin(word: &str) -> String {
    let vowels = ['a', 'e', 'i', 'o', 'u'];
    let first = word.chars().next().unwrap();
    
    if vowels.contains(&first.to_ascii_lowercase()) {
        format!("{}hay", word)
    } else {
        let rest: String = word.chars().skip(1).collect();
        format!("{}{}ay", rest, first)
    }
}
```

</details>

---

## Summary

| Concept | Detail |
|---------|--------|
| `String` | Owned, heap-allocated, growable `Vec<u8>` |
| `&str` | Borrowed slice of UTF-8 bytes |
| UTF-8 | Variable-width: 1-4 bytes per character |
| `.len()` | Returns byte count, NOT character count |
| `.chars().count()` | Returns Unicode scalar count |
| No indexing | `s[0]` is a compile error |
| Slicing | By byte range only, must be at char boundaries |
| Prefer `&str` | For function parameters (accepts both types) |

---

**Previous:** [← Vector Memory Layout](./02-vector-memory-layout.md) · **Next:** [HashMaps →](./04-hashmaps.md)

<p align="center"><i>Tutorial 3 of 8 — Stage 5: Collections</i></p>

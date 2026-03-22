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

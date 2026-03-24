# Rust vs Other Languages 🔄

> **Coming from another language? Here's a side-by-side comparison to help you map what you already know to Rust.**

---

## Table of Contents

- [The Language Spectrum](#the-language-spectrum)
- [Syntax Comparison — At a Glance](#syntax-comparison--at-a-glance)
- [Rust vs C](#rust-vs-c)
- [Rust vs C++](#rust-vs-c-1)
- [Rust vs Python](#rust-vs-python)
- [Rust vs Go](#rust-vs-go)
- [Rust vs JavaScript / TypeScript](#rust-vs-javascript--typescript)
- [Rust vs Java](#rust-vs-java)
- [Concepts That Exist Only in Rust](#concepts-that-exist-only-in-rust)
- [When to Use Rust (and When Not To)](#when-to-use-rust-and-when-not-to)
- [Migration Cheat Sheets](#migration-cheat-sheets)
- [Summary](#summary)

---

## The Language Spectrum

Programming languages sit on a spectrum of **control** vs **convenience**:

```
More Control                                              More Convenience
(closer to hardware)                                      (more abstraction)
     │                                                          │
     ▼                                                          ▼
   Assembly → C → C++ → Rust → Go → Java → C# → JavaScript → Python
                           ▲
                           │
                    Rust is HERE:
               High control AND high convenience
               (this is what makes it special)
```

Most languages force you to choose: low-level control OR high-level convenience. Rust gives you both.

### Understanding the Spectrum: How Languages Talk to Hardware

To understand where languages sit on this spectrum, let's trace what happens when you write `x = 5 + 3`:

**Assembly** (raw CPU instructions):
```nasm
mov eax, 5       ; Put 5 in register EAX
add eax, 3       ; Add 3 to EAX (now EAX = 8)
mov [rbp-4], eax ; Store result at memory address rbp-4
```
You specify the exact CPU registers, the exact memory addresses. Total control, total complexity.

**C** (one thin layer above assembly):
```c
int x = 5 + 3;  // Compiler chooses the register and memory location
```
The compiler translates this to something very close to the assembly above. You still think in terms of memory addresses and pointers, but the compiler handles register allocation.

**Rust** (same performance as C, but with safety):
```rust
let x = 5 + 3;  // Also compiles to nearly the same assembly as C
// But: the compiler also checks ownership, borrowing, and lifetimes
```

**Java** (layer of abstraction — JVM):
```java
int x = 5 + 3;  // Compiles to JVM bytecode: iconst_5, iconst_3, iadd, istore_1
// At runtime, the JVM interprets or JIT-compiles this bytecode
```

**Python** (many layers of abstraction):
```python
x = 5 + 3  # At runtime: creates int object for 5, creates int object for 3,
            # calls __add__ method, creates new int object for 8,
            # creates string "x" in namespace dict, maps "x" → int(8)
```
A simple `5 + 3` in Python involves dictionary lookups, object creation, method dispatch, and reference counting. This is why Python is ~100x slower for arithmetic.

The revolutionary thing about Rust is that it generates the same efficient code as C while providing the kind of safety guarantees normally associated with languages much further right on the spectrum.

---

## Syntax Comparison — At a Glance

### Hello World

| Language | Code |
|----------|------|
| **Rust** | `fn main() { println!("Hello!"); }` |
| **C** | `#include <stdio.h>` + `int main() { printf("Hello!\n"); return 0; }` |
| **C++** | `#include <iostream>` + `int main() { std::cout << "Hello!" << std::endl; }` |
| **Python** | `print("Hello!")` |
| **Go** | `package main` + `import "fmt"` + `func main() { fmt.Println("Hello!") }` |
| **JavaScript** | `console.log("Hello!")` |
| **Java** | `public class Main { public static void main(String[] args) { System.out.println("Hello!"); } }` |

### Variables

```rust
// Rust
let x = 5;           // Immutable by default
let mut y = 10;      // Mutable (you must opt in)
y = 20;              // OK — y is mutable
// x = 10;           // ERROR — x is immutable!
```

```c
// C
int x = 5;           // Mutable by default
const int y = 10;    // Immutable (you must opt in)
```

```python
# Python
x = 5                # Mutable, no type annotation needed
x = "hello"          # Can even change type! (dynamically typed)
```

```go
// Go
x := 5               // Mutable by default, type inferred
var y int = 10       // Explicit type
```

```javascript
// JavaScript
let x = 5;           // Mutable
const y = 10;        // Immutable (but objects/arrays inside are still mutable!)
```

> **Key Rust difference:** Variables are **immutable by default**. You must explicitly write `mut` to make them mutable. This prevents accidental mutation and makes code easier to reason about.

### Functions

```rust
// Rust — types are required for parameters and return value
fn add(a: i32, b: i32) -> i32 {
    a + b   // No semicolon = this is the return value (it's an expression)
}
```

```python
# Python — no type annotations required (but you can add them)
def add(a, b):
    return a + b
```

```go
// Go — types come AFTER the name
func add(a int, b int) int {
    return a + b
}
```

```c
// C — types come BEFORE the name
int add(int a, int b) {
    return a + b;
}
```

```javascript
// JavaScript — no type annotations
function add(a, b) {
    return a + b;
}
```

> **Key Rust difference:** The last expression in a function is implicitly the return value (no `return` keyword needed). Also, types are ALWAYS required in function signatures — Rust doesn't guess.

### If/Else

```rust
// Rust — no parentheses needed around condition, braces required
let x = 5;
if x > 3 {
    println!("Big");
} else {
    println!("Small");
}

// if/else is an EXPRESSION (can return a value!)
let label = if x > 3 { "big" } else { "small" };
```

```python
# Python
x = 5
if x > 3:
    print("Big")
else:
    print("Small")

# Ternary
label = "big" if x > 3 else "small"
```

```javascript
// JavaScript — parentheses required around condition
let x = 5;
if (x > 3) {
    console.log("Big");
} else {
    console.log("Small");
}

// Ternary
let label = x > 3 ? "big" : "small";
```

> **Key Rust difference:** `if/else` is an **expression** — it can produce a value. No need for a separate ternary operator (`? :`).

### Loops

```rust
// Rust
// Standard for loop (iterating over a range)
for i in 0..5 {
    println!("{}", i);  // 0, 1, 2, 3, 4
}

// Iterating over a collection
let names = vec!["Alice", "Bob", "Charlie"];
for name in &names {
    println!("{}", name);
}

// While loop
let mut count = 0;
while count < 5 {
    count += 1;     // No ++ operator in Rust!
}

// Infinite loop (Rust has a dedicated keyword)
loop {
    break;  // Must break or it runs forever
}
```

```python
# Python
for i in range(5):
    print(i)

names = ["Alice", "Bob", "Charlie"]
for name in names:
    print(name)
```

```go
// Go
for i := 0; i < 5; i++ {
    fmt.Println(i)
}
```

> **Key Rust differences:**
> - No C-style `for (int i = 0; i < 5; i++)` — Rust uses ranges: `for i in 0..5`
> - No `++` or `--` operators — use `+= 1` and `-= 1`
> - Rust has a dedicated `loop` keyword for infinite loops (clearer than `while true`)

---

## Rust vs C

### What's Similar

| Feature | C | Rust |
|---------|---|------|
| Compiled to native code | ✅ | ✅ |
| No garbage collector | ✅ | ✅ |
| Manual memory layout control | ✅ | ✅ |
| Can call C libraries | N/A | ✅ (via FFI) |
| Suitable for OS/embedded | ✅ | ✅ |
| Near-identical performance | ✅ | ✅ |

### What's Different

| Feature | C | Rust |
|---------|---|------|
| Memory safety | ❌ Manual (malloc/free) | ✅ Ownership system (compile-time) |
| Null pointers | ❌ `NULL` exists everywhere | ✅ No null — uses `Option<T>` |
| Buffer overflows | ❌ No bounds checking | ✅ Bounds checked by default |
| Data races | ❌ Possible, no protection | ✅ Prevented at compile time |
| Package manager | ❌ None standard | ✅ Cargo |
| Generics | ❌ Void pointers (unsafe) | ✅ Real generics (zero-cost) |
| Error handling | ❌ Return codes (-1, errno) | ✅ Result<T, E> type |
| Standard library | Small | Rich (collections, I/O, threads, etc.) |

### Code Comparison: Reading a File

**C:**
```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    FILE *file = fopen("hello.txt", "r");
    if (file == NULL) {
        perror("Failed to open file");
        return 1;
    }
    
    // Need to allocate buffer, track size, check for errors...
    char buffer[1024];
    size_t bytes = fread(buffer, 1, sizeof(buffer) - 1, file);
    buffer[bytes] = '\0';
    
    printf("%s", buffer);
    fclose(file);  // Must remember to close! (resource leak if you forget)
    return 0;
}
// What if fread fails? What if the file is bigger than 1024 bytes?
// What if you forget fclose()? C doesn't help you.
```

**Rust:**
```rust
use std::fs;

fn main() {
    // read_to_string handles ALL of those edge cases
    match fs::read_to_string("hello.txt") {
        Ok(contents) => println!("{}", contents),
        Err(error) => eprintln!("Failed to open file: {}", error),
    }
    // File is automatically closed when it goes out of scope
    // Buffer is automatically sized
    // All error cases are handled
}
```

---

## Rust vs C++

### What's Similar

Both languages offer:
- Zero-cost abstractions
- Templates/Generics
- RAII (Resource Acquisition Is Initialization)
- Move semantics
- No garbage collector
- Operator overloading
- Pattern matching (C++17 has limited version)

### What's Different

| Feature | C++ | Rust |
|---------|-----|------|
| Memory safety | ❌ Smart pointers help, but not enforced | ✅ Enforced by compiler |
| Undefined behavior | ❌ Easy to trigger accidentally | ✅ Only possible in `unsafe` blocks |
| Header files | ❌ .h/.hpp files, include guards | ✅ Module system, no headers |
| Build system | ❌ CMake, Make, Bazel, etc. (fragmented) | ✅ Cargo (one tool) |
| Package manager | ❌ vcpkg, conan (fragmented) | ✅ Cargo + crates.io |
| Compile times | ❌ Can be very slow | ⚠️ Slow but improving |
| Error messages | ❌ Often cryptic (template errors) | ✅ Famously helpful |
| Inheritance | ✅ Class hierarchies | ❌ No inheritance — uses traits (composition) |
| Null | ❌ nullptr exists | ✅ No null — Option<T> |
| Exceptions | ✅ try/catch | ❌ No exceptions — Result<T, E> |

### The Key Philosophical Difference

```
C++ says: "Here are powerful tools. Try not to misuse them."
    → Trust the programmer. Lots of footguns.

Rust says: "Here are powerful tools. The compiler prevents misuse."
    → Trust the compiler. Very few footguns.
```

---

## Rust vs Python

This is the most common comparison for people learning Rust.

### At a Glance

| Feature | Python | Rust |
|---------|--------|------|
| **Speed** | Slow (10-100x slower) | Fast (as fast as C) |
| **Typing** | Dynamic (types checked at runtime) | Static (types checked at compile time) |
| **Memory** | Garbage collected | Ownership system (compile-time) |
| **Syntax** | Very concise | More verbose (but more explicit) |
| **Learning curve** | Easy | Steeper (but rewarding) |
| **Package manager** | pip | Cargo |
| **Use case** | Scripting, ML, web, prototyping | Systems, CLI, web, performance-critical |
| **Error handling** | Exceptions (try/except) | Result<T, E> (explicit) |
| **Null** | None (can cause AttributeError) | No null — Option<T> (compile-time safe) |
| **Concurrency** | GIL limits parallelism | True parallelism, fearless concurrency |

### Equivalent Code: FizzBuzz

**Python:**
```python
for i in range(1, 101):
    if i % 15 == 0:
        print("FizzBuzz")
    elif i % 3 == 0:
        print("Fizz")
    elif i % 5 == 0:
        print("Buzz")
    else:
        print(i)
```

**Rust:**
```rust
fn main() {
    for i in 1..=100 {
        if i % 15 == 0 {
            println!("FizzBuzz");
        } else if i % 3 == 0 {
            println!("Fizz");
        } else if i % 5 == 0 {
            println!("Buzz");
        } else {
            println!("{}", i);
        }
    }
}
```

Pretty similar! Rust is slightly more verbose (curly braces, `println!` vs `print`), but the logic is identical.

### Performance Difference

A real benchmark — counting primes up to 10 million:

```
Python:  ~45 seconds
Rust:    ~0.3 seconds (150x faster!)
```

This isn't a Rust-specific trick — it's the fundamental difference between an interpreted language with a GIL and a compiled language with native machine code.

#### Why Exactly is Python Slower? — A Deep Dive

The speed difference isn't just "interpreted vs compiled." It's about what happens at every single step:

**Memory layout:**
```
Python integer "42":                Rust i32 "42":
┌─────────────────────┐            ┌──────┐
│ ob_refcnt: 8 bytes  │            │  42  │  ← Just 4 bytes. That's it.
│ ob_type: 8 bytes    │            └──────┘
│ ob_size: 8 bytes    │
│ ob_digit: 4+ bytes  │            Total: 4 bytes
└─────────────────────┘
Total: 28+ bytes for a single number!
```

A Python integer is a full **heap-allocated object** with a reference count, a type pointer, and a size field. Rust's `i32` is just 4 bytes on the stack — no overhead at all.

**Dictionary lookups:**
When Python executes `x = 5`, it does a dictionary lookup to map the string `"x"` to the value. When you access `x`, it does another dictionary lookup. Rust's `x` is just a memory address known at compile time — no lookups needed at runtime.

**Type checking:**
Python checks types at runtime: "Is this thing I'm about to add actually a number?" Rust checks types at compile time and erases them — the CPU just sees memory addresses and arithmetic instructions.

**The GIL (Global Interpreter Lock):**
CPython has a lock that prevents multiple threads from executing Python bytecode simultaneously. This means Python can't truly use multiple CPU cores for computation. Rust has no such limitation — you get real parallelism.

### When to Use Each

| Situation | Use Python | Use Rust |
|-----------|-----------|----------|
| Quick script / prototype | ✅ | ❌ |
| Data science / ML | ✅ (numpy, pandas) | ⚠️ (growing ecosystem) |
| CLI tool (ship to users) | ❌ (needs Python installed) | ✅ (single binary) |
| Web API | ✅ (Django, Flask) | ✅ (higher performance) |
| Performance-critical | ❌ | ✅ |
| Embedded / systems | ❌ | ✅ |
| Raspberry Pi / IoT | ✅ (scripting) | ✅ (performance) |

> **Pro tip:** Many teams use Python AND Rust together. Write the core performance-critical code in Rust, expose it as a Python library via PyO3. This is exactly what `polars`, `tiktoken`, and `huggingface/tokenizers` do.

---

## Rust vs Go

Go and Rust are often compared because they were both created in the late 2000s to solve modern programming challenges. But they made very different design decisions.

### At a Glance

| Feature | Go | Rust |
|---------|-----|------|
| **Speed** | Fast (but not as fast as Rust) | Fastest (tied with C/C++) |
| **Memory** | Garbage collected | Ownership (compile-time) |
| **Learning curve** | Easy | Steeper |
| **Concurrency** | Goroutines (simple) | Threads/async (more control) |
| **Generics** | Added in Go 1.18 (basic) | Advanced generics from the start |
| **Error handling** | `error` interface, `if err != nil` | `Result<T, E>`, `?` operator |
| **Null safety** | ❌ `nil` panics exist | ✅ No null |
| **Compile speed** | Very fast | Slower |
| **Binary size** | Larger (includes runtime) | Smaller |
| **Philosophy** | Simplicity (less features) | Expressiveness (more features) |

### Error Handling Comparison

**Go (verbose — the `if err != nil` pattern):**
```go
file, err := os.Open("hello.txt")
if err != nil {
    return fmt.Errorf("opening file: %w", err)
}
defer file.Close()

data, err := io.ReadAll(file)
if err != nil {
    return fmt.Errorf("reading file: %w", err)
}

fmt.Println(string(data))
```

**Rust (concise — the `?` operator):**
```rust
fn read_file() -> Result<(), Box<dyn std::error::Error>> {
    let contents = std::fs::read_to_string("hello.txt")?;
    println!("{}", contents);
    Ok(())
}
```

The `?` operator in Rust propagates errors automatically. In Go, every function call needs an explicit `if err != nil` check.

### When to Use Each

| Situation | Use Go | Use Rust |
|-----------|--------|----------|
| Simple microservices | ✅ | ✅ |
| Fast compilation matters | ✅ | ❌ |
| Maximum runtime performance | ❌ | ✅ |
| Predictable latency (no GC pauses) | ❌ | ✅ |
| Team with junior developers | ✅ | ⚠️ |
| Systems programming | ❌ | ✅ |
| DevOps tools | ✅ (Docker, K8s) | ✅ |
| WebAssembly | ⚠️ | ✅ |

---

## Rust vs JavaScript / TypeScript

### At a Glance

| Feature | JavaScript/TypeScript | Rust |
|---------|----------------------|------|
| **Runtime** | Node.js / Browser (V8, SpiderMonkey) | Compiled to native code |
| **Typing** | Dynamic (JS) / Static (TS) | Static, much stricter than TS |
| **Speed** | Moderate (JIT compiled) | Very fast (AOT compiled) |
| **Memory** | Garbage collected | Ownership (compile-time) |
| **Null safety** | ❌ `null`/`undefined` | ✅ `Option<T>` |
| **Package manager** | npm/yarn/pnpm | Cargo |
| **Error handling** | try/catch + Promises | Result<T, E> |
| **Concurrency** | Single-threaded event loop | Multi-threaded |
| **Package count** | 2M+ (npm) | 140K+ (crates.io) |

### Code Comparison: Filter and Map

**JavaScript:**
```javascript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
const result = numbers
    .filter(n => n % 2 === 0)
    .map(n => n * n);
console.log(result); // [4, 16, 36, 64, 100]
```

**Rust:**
```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let result: Vec<i32> = numbers
        .iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * n)
        .collect();
    println!("{:?}", result); // [4, 16, 36, 64, 100]
}
```

Similar functional style! Rust requires slightly more syntax (`iter()`, `collect()`, type annotation) but produces code that runs 10-50x faster.

### Where Rust Overlaps with JS/TS

- **WebAssembly (Wasm)**: Rust compiles to Wasm — you can run Rust code in the browser! Libraries like `wasm-bindgen` make this easy.
- **Tauri**: Build desktop apps with a Rust backend and web frontend (alternative to Electron, much smaller binaries).
- **Deno**: The Deno JavaScript runtime is built in Rust.

---

## Rust vs Java

### At a Glance

| Feature | Java | Rust |
|---------|------|------|
| **Runtime** | JVM (virtual machine) | Native machine code |
| **Speed** | Good (JIT optimized) | Excellent (AOT compiled) |
| **Memory** | Garbage collected | Ownership (compile-time) |
| **OOP** | Class-based inheritance | Trait-based composition |
| **Null** | ❌ NullPointerException | ✅ Option<T> — no null |
| **Exceptions** | try/catch | Result<T, E> |
| **Boilerplate** | Lots (getters, setters, etc.) | Less (but explicit types) |
| **Build tool** | Maven/Gradle | Cargo |
| **Deployment** | Needs JVM installed | Single binary |

### The Null Safety Difference

**Java (NullPointerException at runtime):**
```java
String name = getUserName();  // Might return null!
System.out.println(name.length());  // 💥 NullPointerException if null!
```

**Rust (caught at compile time):**
```rust
let name: Option<String> = get_user_name();  // Explicitly MIGHT be absent
// name.len();  // ❌ Won't compile! Must handle the None case first.

match name {
    Some(n) => println!("Length: {}", n.len()),
    None => println!("No name provided"),
}
```

In Rust, if a value might be absent, it MUST be `Option<T>`, and you MUST handle both `Some` and `None` cases. The billion-dollar mistake (null) simply doesn't exist in Rust.

---

## Concepts That Exist Only in Rust

These are things you won't find (in the same way) in other languages:

| Concept | What It Is | Why It's Unique |
|---------|-----------|-----------------|
| **Ownership** | Every value has exactly one owner | Eliminates whole classes of memory bugs |
| **Borrowing** | References can borrow values | Allows sharing without copying |
| **Borrow Checker** | Compiler verifies borrows are safe | No runtime cost, compile-time guarantee |
| **Lifetimes** | Annotations describing reference validity | Prevents dangling references at compile time |
| **Zero-cost abstractions** | High-level features with no runtime overhead | Iterators are as fast as manual loops |
| **Algebraic data types** | Enums with associated data | Better than C unions + tag |
| **Exhaustive matching** | Match must handle every case | No missed cases for enums |
| **No null** | Option<T> instead of null | No NullPointerException. Ever. |
| **Fearless concurrency** | Data race prevention at compile time | If it compiles, no data races |
| **Cargo** | Build, package, test, format, lint — one tool | No fragmented tooling ecosystem |

We'll learn ALL of these concepts in the upcoming stages!

### The "Billion Dollar Mistake" — Why No Null Matters

In 1965, **Sir Tony Hoare** invented null references for the ALGOL W language. In 2009, he called it his "billion dollar mistake":

> "I call it my billion-dollar mistake. It was the invention of the null reference in 1965... This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years."

Every language that inherited null (C, C++, Java, C#, JavaScript, Python `None`, Go `nil`) inherited this problem. The issue: you can never be sure a value isn't null without checking, and the compiler doesn't force you to check.

Rust's solution (`Option<T>`) is based on ideas from **ML** (1973) and **Haskell** (1990), where the concept of "a value that might not exist" is represented as a type:

```
Haskell:    Maybe a = Nothing | Just a
OCaml:      type 'a option = None | Some of 'a  
Rust:       enum Option<T> { None, Some(T) }
```

The key insight: by making "absence" part of the type system, the compiler can force you to handle both cases. You literally cannot access the inner value without first checking that it exists. NullPointerException becomes impossible.

### Where Do These Concepts Come From? — A Family Tree

```
1960s:  LISP (garbage collection, closures)
  │
1970s:  ML (pattern matching, algebraic types, type inference)
  │     C (manual memory, pointers, systems programming)
  │
1980s:  C++ (RAII, zero-cost abstractions, move semantics)
  │     Haskell (typeclasses → Rust traits, monads → Result/Option)
  │
1990s:  Java (GC, no pointer arithmetic)
  │     OCaml (practical ML with fast compilation)
  │
2000s:  Cyclone (region-based memory → Rust lifetimes)
  │     C++ smart pointers (unique_ptr → Rust ownership)
  │
2006:   Graydon Hoare starts combining ALL the above into Rust
  │
2015:   Rust 1.0 — the synthesis is complete
```

Rust is a **synthesis language** — it combines proven ideas from 50 years of programming language research into a single, practical language. The genius isn't in any single feature, but in how they all fit together to provide safety AND performance simultaneously.

---

## When to Use Rust (and When Not To)

### Use Rust When

- **Performance matters** — Games, databases, network services, CLI tools
- **Reliability matters** — Medical devices, financial systems, infrastructure
- **Resource constrained** — Embedded, IoT, WebAssembly
- **Concurrency needed** — Multi-threaded servers, parallel computation
- **Security critical** — Cryptography, authentication, parsers for untrusted input
- **Ship as a binary** — No runtime dependency (unlike Java, Python, Node)
- **Fun and learning** — Rust makes you a better programmer in any language

### Consider Other Languages When

- **Rapid prototyping** — Python, JavaScript (faster to write)
- **Data science** — Python (pandas, numpy, scikit-learn ecosystem)
- **Mobile apps** — Swift (iOS), Kotlin (Android) have better platform integration
- **Simple web apps** — Any language works, use what your team knows
- **Scripting** — Python, Bash (lower overhead for small scripts)

### The Pragmatic Answer

> **Use the best tool for the job.** Rust is outstanding for many use cases, but it's not always the right choice. The best developers know multiple languages and pick the right one for each project.

---

## Migration Cheat Sheets

### If You Know Python

| Python | Rust |
|--------|------|
| `x = 5` | `let x = 5;` |
| `x = 5` (reassign) | `let mut x = 5; x = 10;` |
| `def foo(x: int) -> int:` | `fn foo(x: i32) -> i32 {` |
| `print(f"Hello {name}")` | `println!("Hello {}", name);` |
| `for i in range(10):` | `for i in 0..10 {` |
| `if x > 3:` | `if x > 3 {` |
| `my_list = [1, 2, 3]` | `let my_vec = vec![1, 2, 3];` |
| `my_dict = {"a": 1}` | `let my_map = HashMap::from([("a", 1)]);` |
| `None` | `None` (inside `Option<T>`) |
| `try/except` | `match result { Ok(v) => ..., Err(e) => ... }` |
| `class Foo:` | `struct Foo { } impl Foo { }` |
| `import os` | `use std::fs;` |

### If You Know JavaScript / TypeScript

| JavaScript/TypeScript | Rust |
|-----------------------|------|
| `let x = 5` | `let x = 5;` |
| `const x = 5` | `let x = 5;` (immutable by default!) |
| `let x: number = 5` | `let x: i32 = 5;` |
| `` `Hello ${name}` `` | `format!("Hello {}", name)` |
| `[1, 2, 3]` | `vec![1, 2, 3]` |
| `arr.map(x => x * 2)` | `arr.iter().map(\|x\| x * 2).collect()` |
| `arr.filter(x => x > 3)` | `arr.iter().filter(\|x\| **x > 3).collect()` |
| `null` / `undefined` | `None` (inside `Option<T>`) |
| `interface / type` | `trait` / `struct` |
| `async / await` | `async / .await` (note the dot!) |

---

## Summary

| Language | Rust's Advantage | That Language's Advantage |
|----------|-----------------|--------------------------|
| **C** | Memory safety, no undefined behavior, Cargo | Simpler, smaller, more embedded support |
| **C++** | Safer, better tooling, no header files | Larger ecosystem, more mature |
| **Python** | 10-100x faster, type safety, no GIL | Faster to write, ML ecosystem |
| **Go** | Faster, no GC pauses, richer type system | Faster compilation, simpler to learn |
| **JS/TS** | Much faster, real concurrency, null safety | Web platform, huge npm ecosystem |
| **Java** | No JVM, smaller binaries, null safety | More enterprise libraries, wider adoption |

### Key Takeaway

> Rust occupies a unique position: the performance and control of C/C++ with the safety and developer experience of modern high-level languages. Its ownership model is genuinely new — no other mainstream language has it. Learning Rust will make you a better programmer regardless of what language you use day to day.

---

## 🎉 Stage 1 Complete!

Congratulations! You've finished **Stage 1: The Starting Line**. You now know:

- ✅ Why Rust exists and what problems it solves
- ✅ How to install Rust and set up your editor
- ✅ How to write, compile, and run a Rust program
- ✅ How Cargo works — Rust's incredible build system
- ✅ The Rust tooling ecosystem (playground, rustfmt, clippy, rust-analyzer)
- ✅ How Rust compares to languages you already know

### What's Next?

It's time to learn the actual language! In Stage 2, we'll cover variables, types, functions, and control flow — the fundamental building blocks of every Rust program.

**Next Stage:** [Stage 2: Rust Fundamentals →](../02-rust-fundamentals/)

---

<p align="center">
  <i>Tutorial 6 of 6 — Stage 1: The Starting Line</i>
</p>

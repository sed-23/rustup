# Hello, World! 👋

> **Your very first Rust program. We'll explain every single character — nothing is magic.**

---

## Table of Contents

- [Two Ways to Run Rust](#two-ways-to-run-rust)
- [Method 1: Using rustc Directly](#method-1-using-rustc-directly)
- [Dissecting Every Character](#dissecting-every-character)
- [Method 2: Using Cargo (The Better Way)](#method-2-using-cargo-the-better-way)
- [What Happened Behind the Scenes?](#what-happened-behind-the-scenes)
- [From Source Code to Executable](#from-source-code-to-executable)
- [Playing with println!](#playing-with-println)
- [Your First Errors (On Purpose!)](#your-first-errors-on-purpose)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Two Ways to Run Rust

There are two ways to compile and run Rust code:

| Method | When to Use |
|--------|-------------|
| `rustc` (compiler directly) | Single-file experiments, understanding the basics |
| `cargo` (build system) | Everything else — ALL real projects use Cargo |

We'll do both, so you understand what's happening under the hood.

---

## Method 1: Using rustc Directly

### Step 1: Create a File

Open your terminal and create a folder for your Rust experiments:

```bash
# Create a folder
mkdir rust-hello
cd rust-hello
```

Now create a file called `main.rs`:

**On macOS/Linux:**
```bash
touch main.rs
```

**On Windows (PowerShell):**
```powershell
New-Item main.rs
```

Or just create it in your editor (VS Code: `code main.rs`).

### Step 2: Write the Code

Open `main.rs` in your editor and type this:

```rust
fn main() {
    println!("Hello, world!");
}
```

> ⚠️ **Type it out — don't copy-paste!** Typing builds muscle memory and helps you notice the details.

### Step 3: Compile It

In your terminal (make sure you're in the `rust-hello` folder):

```bash
rustc main.rs
```

This creates an executable:
- On **Windows**: `main.exe`
- On **macOS/Linux**: `main`

### Step 4: Run It

**Windows:**
```powershell
.\main.exe
```

**macOS/Linux:**
```bash
./main
```

### Output:

```
Hello, world!
```

🎉 **Congratulations!** You just wrote, compiled, and ran your first Rust program!

### The History of "Hello, World!"

The tradition of "Hello, World!" as the first program goes back to **1978**, when Brian Kernighan and Dennis Ritchie published *The C Programming Language*. The very first example in the book was:

```c
#include <stdio.h>
main() {
    printf("hello, world\n");
}
```

Since then, virtually every programming language tutorial starts with "Hello, World!" It serves as a **smoke test** — if you can get this to work, your entire toolchain (editor, compiler, linker, runtime) is functioning correctly.

Fun fact: Kernighan actually used "hello, world" (lowercase, no exclamation mark). Over time, it evolved into "Hello, World!" or "Hello, world!" depending on the language community.

---

## Dissecting Every Character

Let's break down `main.rs` character by character. Nothing is magic — everything has a purpose.

```rust
fn main() {
    println!("Hello, world!");
}
```

### Line 1: `fn main() {`

```
fn        → Keyword that means "I'm defining a function"
main      → The name of the function. "main" is SPECIAL in Rust.
()        → The parameter list. Empty = this function takes no inputs.
{         → Opening curly brace. The function body starts here.
```

#### What's Special About `main`?

Every Rust program starts by running the `main` function. It's the **entry point** — the very first thing that executes when you run your program.

```
When you run ./main:
  1. The operating system loads your program into memory
  2. It looks for the "main" function
  3. It starts executing whatever's inside main()
  4. When main() ends, the program is done
```

#### How Entry Points Work Under the Hood

When you double-click an executable or run it from a terminal, here's the full journey:

1. **The OS loader** reads the binary's headers (ELF on Linux, PE on Windows, Mach-O on macOS) to find the **entry point address** — this is NOT `main` directly
2. **The C runtime (crt0)** runs first. It sets up the stack, initializes global variables, and sets up signal handlers
3. **Rust's runtime initialization** runs — this is minimal (unlike Java's JVM startup or Python's interpreter initialization). Rust sets up a panic handler, initializes the thread system, and parses command-line arguments
4. **Your `main()` function** is finally called
5. When `main()` returns, cleanup code runs (`Drop` implementations), and the process exits

This is why even a simple Rust binary is a few hundred kilobytes — it includes the C runtime, Rust's minimal runtime, and the standard library components you used. But compared to languages like Go (which includes an entire garbage collector) or Java (which needs the JVM), this overhead is tiny.

> This is the same as C and C++. If you know Python, it's like the code at the top level of your script. If you know JavaScript, think of it like the code that runs when you execute `node index.js`.

#### What Entry Points Look Like in Other Languages

```python
# Python — no explicit entry point, code runs top-to-bottom
print("Hello!")

# Convention (not required):
if __name__ == "__main__":
    print("Hello!")
```

```java
// Java — entry point must be a static method in a class
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello!");
    }
}
```

```go
// Go — must be in package main
package main
func main() {
    fmt.Println("Hello!")
}
```

```c
// C — the original. Rust follows this convention
int main(void) {
    printf("Hello!\n");
    return 0;  // Return code (0 = success)
}
```

Notice that Rust's `main()` looks most like C's, but without the `int` return type and `return 0`. Rust's `main` returns `()` (the unit type, meaning "nothing") by default. Later, you'll learn you can make `main` return `Result` for error handling.

#### Rules for `main`:

- Every executable Rust program MUST have exactly one `main` function
- By default, `main` takes no arguments and returns nothing
- (Later you'll learn it can return `Result` for error handling)

### Line 2: `println!("Hello, world!");`

```
println!  → A MACRO (not a function!) that prints text and a newline
(         → Opening parenthesis for the macro's arguments
"         → Start of a string literal
Hello, world!  → The text to print
"         → End of the string literal
)         → Closing parenthesis
;         → Semicolon — this statement is done
```

#### Wait, What's a Macro?

See that `!` after `println`? That means `println!` is a **macro**, not a regular function.

```
println!     → Macro (has !)
println      → This would be a function call (no !)
```

**For now**, think of macros as "special functions that can do extra things." The reason `println!` is a macro is that it can accept a variable number of arguments and format them:

```rust
println!("Hello!");                              // 1 argument
println!("Hello, {}!", "world");                 // 2 arguments
println!("{} + {} = {}", 1, 2, 3);              // 4 arguments
```

A regular Rust function can't do this (function signatures are fixed). Macros can generate code at compile time. We'll cover macros in depth in Stage 21. For now, just remember: **`!` = macro**.

#### Why Macros Instead of Variadic Functions?

In C, `printf` is a variadic function — it accepts any number of arguments:
```c
printf("%s is %d years old\n", "Alice", 30);
```

But C's variadic functions are **type-unsafe**. If you write `printf("%d", "hello")`, C will happily compile it and produce garbage output (or crash). The format string says "integer" but you passed a string — C doesn't check.

Rust's `println!` macro, on the other hand, **checks format strings at compile time**:
```rust
println!("{}", 42);        // ✅ Compiles
// println!("{} {}", 42);  // ❌ Compile error: 2 placeholders but 1 argument
```

This is only possible because `println!` is a macro — it can inspect the format string during compilation and verify that the number and types of arguments match. A regular function cannot do this.

#### What the Macro Actually Generates

When you write:
```rust
println!("Hello, {}!", "world");
```

The macro expands (at compile time) to something roughly like:
```rust
{
    ::std::io::_print(
        ::core::fmt::Arguments::new_v1(
            &["Hello, ", "!\n"],
            &[::core::fmt::ArgumentV1::new(&"world", ::core::fmt::Display::fmt)]
        )
    );
}
```

You never need to write this — the macro does it for you. But it's good to know that macros are just code that writes code.

#### The String `"Hello, world!"`

- Text between double quotes `" "` is a **string literal**
- Rust strings are **UTF-8 encoded** (they support emojis, Chinese characters, etc.)
- String literals are stored in the program's binary (we'll explore this in Stage 3)

#### Where String Literals Live in Memory

When you write `"Hello, world!"` in your code, the text is baked directly into the compiled binary — it becomes part of the program's **read-only data segment** (also called `.rodata` on Linux or `.rdata` on Windows). Here's the memory layout:

```
Your compiled binary on disk:
┌──────────────────────┐
│   Code (.text)       │  ← Your compiled functions
├──────────────────────┤
│   Read-only data     │  ← "Hello, world!" lives HERE
│   (.rodata)          │    It's embedded in the binary forever
├──────────────────────┤
│   Other sections     │
└──────────────────────┘

At runtime:
┌──────────────────────┐
│   Code segment       │  ← Instructions the CPU executes
├──────────────────────┤
│   Data segment       │  ← "Hello, world!" is loaded here
│   (read-only)        │    Address: e.g., 0x00401000
├──────────────────────┤
│   Stack              │  ← Local variables  
├──────────────────────┤
│   Heap               │  ← Dynamic allocations (Vec, String, Box)
└──────────────────────┘
```

The variable in your code is actually a **pointer** (memory address + length) to this read-only data. The type `&str` literally means "a reference to string data stored somewhere." For string literals, that "somewhere" is the binary itself. This is why string literals have the `'static` lifetime — they exist for the entire duration of the program.

This is the same approach C uses for string literals, and it means string literals are essentially **free** — no heap allocation, no cleanup needed. The operating system loads them into memory when the program starts and unmaps them when it ends.

#### The Semicolon `;`

In Rust, most statements end with a semicolon. This is different from Python (no semicolons) but similar to C, C++, Java, and JavaScript.

```rust
println!("Hello");   // ← Semicolon! This is a statement.
```

> **Not every line needs a semicolon** — we'll learn about expressions vs statements in Stage 2. For now, put a semicolon at the end of every line.

### Line 3: `}`

```
}  → Closing brace. The function body ends here.
```

The `{` on line 1 and the `}` on line 3 are a **pair** — they define the block of code that belongs to the `main` function.

### The Complete Picture

```rust
fn main() {                      // Define the entry point function
    println!("Hello, world!");   // Print text to the console
}                                // End of the function
```

---

## Method 2: Using Cargo (The Better Way)

Using `rustc` directly is fine for one file, but **real Rust projects use Cargo**. Let's do the same thing the proper way.

### Step 1: Create a New Project

```bash
cargo new hello_world
```

Output:
```
    Creating binary (application) `hello_world` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
```

### Step 2: Look at What Cargo Created

```bash
cd hello_world
```

Cargo created this structure:

```
hello_world/
  ├── Cargo.toml     ← Project configuration (like package.json in Node.js)
  └── src/
        └── main.rs   ← Your source code
```

And it **already wrote Hello World for you!** Let's check:

```bash
cat src/main.rs
```

```rust
fn main() {
    println!("Hello, world!");
}
```

Cargo also initialized a **git repository** for you (`hello_world/.git`).

### Step 3: Build and Run (Two Separate Commands)

**Build (compile):**
```bash
cargo build
```

Output:
```
   Compiling hello_world v0.1.0 (/path/to/hello_world)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.50s
```

The compiled binary is at `target/debug/hello_world` (or `target\debug\hello_world.exe` on Windows).

**Run the compiled binary:**
```bash
./target/debug/hello_world
```

Output:
```
Hello, world!
```

### Step 4: Build and Run in One Command

There's a shortcut that builds AND runs:

```bash
cargo run
```

Output:
```
   Compiling hello_world v0.1.0 (/path/to/hello_world)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.30s
     Running `target/debug/hello_world`
Hello, world!
```

> **`cargo run` is the command you'll use 99% of the time during development.**

### Step 5: Check Without Running

```bash
cargo check
```

This checks if your code compiles BUT doesn't produce a binary. It's **faster** than `cargo build` — useful for quickly checking your code during development.

### Cargo Commands Summary

| Command | What It Does |
|---------|-------------|
| `cargo new project_name` | Create a new project |
| `cargo build` | Compile the project |
| `cargo run` | Compile AND run the project |
| `cargo check` | Check for errors without compiling fully |
| `cargo build --release` | Compile with optimizations (for production) |

---

## What Happened Behind the Scenes?

When you ran `cargo build`, here's what happened step by step:

```
1. Cargo reads Cargo.toml to understand the project
2. Cargo calls rustc with the right flags
3. rustc reads src/main.rs
4. rustc parses your code into an AST (Abstract Syntax Tree)
5. rustc checks for errors:
   - Syntax errors (typos, missing semicolons)
   - Type errors (wrong types)
   - Borrow checker (ownership violations) — Rust's special sauce!
6. If all checks pass, rustc generates machine code
7. The linker creates the final executable
8. The binary is placed in target/debug/
```

### Debug vs Release Builds

| Build | Command | Speed of Compilation | Speed of Program | Use When |
|-------|---------|---------------------|-------------------|----------|
| **Debug** | `cargo build` | Fast compile | Slower runtime | During development |
| **Release** | `cargo build --release` | Slower compile | Fastest runtime | For production/benchmarks |

Debug builds include extra information for debugging and skip optimizations, so they compile fast but run slower. Release builds spend more time optimizing, so they compile slower but your program runs much faster.

```bash
# Debug build (fast to compile, slow to run)
cargo build
# Binary: target/debug/hello_world

# Release build (slow to compile, fast to run)
cargo build --release
# Binary: target/release/hello_world
```

---

## From Source Code to Executable

Here's a visual representation of what the compiler does:

```
┌─────────────────────┐
│     main.rs          │    Your source code (text file)
│ fn main() {          │
│   println!("...");   │
│ }                    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Lexer / Parser     │    Breaks code into tokens, builds syntax tree
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Type Checking      │    Are all types correct?
│   Borrow Checking    │    Are ownership rules satisfied?
└─────────┬───────────┘
          │              ← If errors found, compilation STOPS here
          ▼                (with helpful error messages!)
┌─────────────────────┐
│   MIR (Mid-Level IR) │    Intermediate representation
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   LLVM IR            │    Lower-level intermediate representation
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   LLVM Backend       │    Generates actual machine code
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Linker             │    Links with system libraries
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Executable Binary  │    The final program you can run!
│   (main or main.exe) │
└─────────────────────┘
```

> **LLVM** is a compiler infrastructure project. Rust (and also Clang/C++, Swift, and Julia) use LLVM to generate optimized machine code. You don't need to know about LLVM to use Rust.

### Why LLVM Matters — A Brief History

LLVM (originally "Low Level Virtual Machine") was created by **Chris Lattner** as a research project at the University of Illinois in 2000. The key idea: instead of every language writing its own code generator for every CPU architecture, languages could compile to LLVM's intermediate representation (IR), and LLVM would handle the optimization and machine code generation.

This is why Rust can run on so many platforms (x86-64, ARM, RISC-V, WebAssembly, etc.) without the Rust team writing a code generator for each one — LLVM handles the last mile.

```
Before LLVM era:
  Each language × each CPU = separate code generator needed
  10 languages × 5 architectures = 50 code generators

With LLVM:
  Each language → LLVM IR → each CPU
  10 languages × 1 IR + 5 backends = 15 implementations total
```

Languages using LLVM: Rust, C/C++ (via Clang), Swift, Julia, Kotlin/Native, Crystal, Zig, and many more. They all benefit from LLVM's decades of optimization work.

### How Different Languages Compare: Source to Execution

```
C / C++ / Rust:
  Source → Compiler → Machine Code → CPU executes directly
  💨 Fastest. No middleman at runtime.

Java / C#:
  Source → Compiler → Bytecode → JVM/CLR interprets or JIT-compiles → CPU
  🏃 Fast after warmup (JIT learns hot paths). Startup is slower.

Python / Ruby:
  Source → Interpreter reads line by line → Executes via C runtime
  🚶 Slowest for compute. Great for glue code and scripting.

JavaScript:
  Source → V8/SpiderMonkey parses → JIT compiles hot functions → CPU
  🏃 Surprisingly fast thanks to aggressive JIT. Still has GC pauses.
```

Rust's approach means your program starts fast (no JIT warmup), runs fast (native code), and exits cleanly (deterministic cleanup). There is no "warm up" period like Java, no interpreter overhead like Python.

---

## Playing with println!

Now that you have a project, let's experiment with `println!`.

### Basic Printing

```rust
fn main() {
    println!("Hello, world!");
    println!("I'm learning Rust!");
    println!("This is my first program.");
}
```

Output:
```
Hello, world!
I'm learning Rust!
This is my first program.
```

### Printing Numbers

```rust
fn main() {
    println!("I am {} years old.", 25);
    println!("{} + {} = {}", 2, 3, 5);
    println!("Pi is approximately {}", 3.14159);
}
```

Output:
```
I am 25 years old.
2 + 3 = 5
Pi is approximately 3.14159
```

The `{}` is a **placeholder**. Rust replaces each `{}` with the next argument, in order.

### Printing Variables

```rust
fn main() {
    let name = "Rustacean";
    let age = 1;
    println!("Hello, {}! You are {} days old.", name, age);
}
```

Output:
```
Hello, Rustacean! You are 1 days old.
```

### print! vs println!

```rust
fn main() {
    print!("Hello ");    // No newline at the end
    print!("World");     // Continues on same line
    println!("!");       // Adds a newline at the end
    println!("New line starts here.");
}
```

Output:
```
Hello World!
New line starts here.
```

| Macro | Does What |
|-------|-----------|
| `print!` | Prints text, NO newline at the end |
| `println!` | Prints text WITH a newline at the end |
| `eprint!` | Prints to stderr, NO newline |
| `eprintln!` | Prints to stderr WITH newline |

### Escape Characters

```rust
fn main() {
    println!("She said \"hello\"");      // \" for quotes inside a string
    println!("Line 1\nLine 2");          // \n for newline
    println!("Column1\tColumn2");        // \t for tab
    println!("Backslash: \\");           // \\ for a literal backslash
    println!("Emoji: \u{1F980}");        // \u{...} for Unicode. 1F980 = 🦀
}
```

Output:
```
She said "hello"
Line 1
Line 2
Column1	Column2
Backslash: \
Emoji: 🦀
```

### Formatting Tricks

```rust
fn main() {
    // Named placeholders
    println!("{name} is {age}", name = "Ferris", age = 5);

    // Debug format (for developers — shows type-specific details)
    println!("{:?}", (1, 2, 3));        // Debug format a tuple
    println!("{:#?}", vec![1, 2, 3]);   // Pretty-printed debug format

    // Number formatting
    println!("Binary: {:b}", 42);       // Binary: 101010
    println!("Hex:    {:x}", 255);      // Hex:    ff
    println!("Octal:  {:o}", 8);        // Octal:  10

    // Padding and alignment
    println!("{:>10}", "right");        //      right  (right-aligned, width 10)
    println!("{:<10}", "left");         // left        (left-aligned, width 10)
    println!("{:^10}", "center");       //   center    (centered, width 10)
    println!("{:0>5}", 42);             // 00042       (zero-padded, width 5)
}
```

Don't worry about memorizing all of these — you'll pick them up naturally as you use them.

---

## Your First Errors (On Purpose!)

Let's intentionally break things to see how Rust's error messages work. This is important because you'll see errors a lot as a beginner, and Rust's error messages are **famously helpful**.

### Error 1: Missing Semicolon

```rust
fn main() {
    println!("Hello, world!")
}
```

Compiler output:
```
error: expected `;`
 --> src/main.rs:2:30
  |
2 |     println!("Hello, world!")
  |                              ^ help: add `;` here
```

Notice how the compiler:
1. Tells you exactly **what** is wrong ("expected `;`")
2. Points to **where** the error is (line 2, column 30)
3. Suggests **how to fix it** ("add `;` here")

### Error 2: Missing Closing Brace

```rust
fn main() {
    println!("Hello, world!");
```

```
error: this file contains an unclosed delimiter
 --> src/main.rs:3:1
  |
1 | fn main() {
  |           - unclosed delimiter
```

### Error 3: Misspelling println

```rust
fn main() {
    printl!("Hello, world!");
}
```

```
error: cannot find macro `printl` in this scope
 --> src/main.rs:2:5
  |
2 |     printl!("Hello, world!");
  |     ^^^^^^ help: a macro with a similar name exists: `println`
```

See that? Rust even **suggests the correct name** when you have a typo. That's amazing error messages.

### Error 4: Missing main Function

```rust
fn hello() {
    println!("Hello!");
}
```

```
error[E0601]: `main` function not found in crate `hello_world`
 --> src/main.rs:1:1
  |
1 | / fn hello() {
2 | |     println!("Hello!");
3 | | }
  | |_^ consider adding a `main` function to `src/main.rs`
```

### The Lesson

> Rust's compiler is your **best friend**, not your enemy. It catches bugs early and tells you exactly how to fix them. When you see an error, read it carefully — the answer is usually right there.

---

## Exercises

### Exercise 1: Personal Greeting

Write a program that prints:
```
Hi, my name is [YOUR NAME].
I am learning Rust today!
Today's date is March 18, 2026.
```

### Exercise 2: ASCII Art

Use `println!` to print this crab:

```
    🦀
   /||\
  / || \
    ||
   /  \
  /    \
```

(Hint: You'll need `\\` to print backslashes)

### Exercise 3: Formatted Info Card

Print this formatted card using `println!` with placeholders `{}`:

```
╔══════════════════════════╗
║  Name:     Ferris        ║
║  Species:  Crab          ║
║  Language: Rust           ║
║  Age:      9 years       ║
╚══════════════════════════╝
```

### Exercise 4: Experiment!

Try these and observe what happens:
1. What if you have TWO `main` functions?
2. What if `main` is spelled `Main` (capital M)?
3. What if you remove the `fn` keyword?
4. What if you use single quotes `'Hello'` instead of double quotes?

---

## Summary

| Concept | Detail |
|---------|--------|
| **Entry point** | Every Rust program starts at `fn main()` |
| **println!** | Macro that prints a line to the console |
| **`!`** | Means it's a macro, not a function |
| **`{}`** | Placeholder in format strings |
| **`;`** | Semicolons end most statements |
| **rustc** | Compiles a single .rs file directly |
| **cargo run** | Creates a project, compiles, and runs — use this! |
| **cargo check** | Checks for errors without making a binary (faster) |

### Key Takeaway

> You've written, compiled, and run your first Rust program. You understand every character in `fn main() { println!("Hello, world!"); }`. From now on, we use Cargo for everything.

---

## What's Next?

Now that you've run your first program, let's explore Cargo in depth — Rust's incredible build system and package manager.

**Next Tutorial:** [Understanding Cargo →](./04-understanding-cargo.md)

---

<p align="center">
  <i>Tutorial 3 of 6 — Stage 1: The Starting Line</i>
</p>

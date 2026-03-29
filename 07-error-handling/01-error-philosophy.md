# Error Philosophy in Rust 🎯

> **Rust doesn't have exceptions. Instead, errors are values — visible in function signatures, enforced by the compiler, and impossible to accidentally ignore. This is not a limitation; it's a superpower.**

---

## Table of Contents

- [The Two Categories of Errors](#the-two-categories-of-errors)
- [Why No Exceptions?](#why-no-exceptions)
- [The Exception Problem in Other Languages](#the-exception-problem-in-other-languages)
- [Rust's Approach: Errors as Values](#rusts-approach-errors-as-values)
- [The Error Trait](#the-error-trait)
- [A Mental Model: When to Panic vs Result](#a-mental-model-when-to-panic-vs-result)
- [Historical Context: The Evolution of Error Handling](#historical-context-the-evolution-of-error-handling)
- [Exercises](#exercises)
- [Summary](#summary)

---

## The Two Categories of Errors

Every program encounters errors. A file doesn't exist. A network connection drops. A user
types "banana" when you expected a number. The question every language must answer is:
**how do you handle these situations?**

Rust splits all errors into exactly two categories:

```
┌─────────────────────────────────────────────────────────────────┐
│                      ERRORS IN RUST                             │
├─────────────────────────────┬───────────────────────────────────┤
│     RECOVERABLE             │       UNRECOVERABLE               │
│                             │                                   │
│  "Something went wrong,     │  "Something is fundamentally      │
│   but we can handle it."    │   broken. We must stop NOW."      │
│                             │                                   │
│  Examples:                  │  Examples:                        │
│  • File not found           │  • Index out of bounds            │
│  • Invalid user input       │  • Assertion failure              │
│  • Network timeout          │  • Integer overflow (debug)       │
│  • Permission denied        │  • Accessing None on unwrap()     │
│  • Parse error              │  • Violated invariants            │
│                             │                                   │
│  Mechanism: Result<T, E>    │  Mechanism: panic!                │
│  You HANDLE the error.      │  The program STOPS.               │
└─────────────────────────────┴───────────────────────────────────┘
```

### Recoverable Errors — `Result<T, E>`

A recoverable error is one where the program can **reasonably continue** after the error
occurs. The operation failed, but the program itself isn't broken:

```rust
use std::fs;

fn main() {
    // Reading a file might fail — and that's okay!
    let content = fs::read_to_string("config.toml");

    match content {
        Ok(text) => println!("Config loaded: {} bytes", text.len()),
        Err(e) => println!("No config file found, using defaults: {}", e),
    }

    // The program continues running normally
    println!("Application started!");
}
```

The function `fs::read_to_string` returns a `Result<String, io::Error>`, not a
`String`. This tells the caller: "Hey, this operation might fail. You need to decide
what to do about it." The type system itself communicates the possibility of failure.

### Unrecoverable Errors — `panic!`

An unrecoverable error means there's a **bug in the program**. Something happened that
should be logically impossible, or the program has entered a state it cannot continue from:

```rust
fn main() {
    let v = vec![1, 2, 3];

    // This is a BUG — we're accessing an index that doesn't exist
    // Rust panics to prevent reading garbage memory
    println!("{}", v[99]);  // 💥 panic! index out of bounds
}
```

When Rust panics, it:
1. Prints an error message
2. Unwinds the stack (cleaning up resources)
3. Terminates the program (or just the thread)

This is intentional. If you're accessing index 99 of a 3-element vector, your program
has a **logic bug**. There's no sensible way to "recover" — the program's assumptions
are wrong.

### The Real-World Analogy

Think of it like driving a car:

```
Recoverable Error (Result):
  You encounter a road closure.
  → You reroute. You keep driving. Life goes on.

Unrecoverable Error (panic!):
  Your engine explodes.
  → You STOP. You don't keep driving on a destroyed engine.
     You call a tow truck (the developer fixes the bug).
```

### Why Two Categories?

Many languages blur this line. In Python, "file not found" and "index out of bounds"
are both exceptions. A `try`/`except` can catch either one. This leads to a dangerous
pattern: developers catch "too much," accidentally swallowing bugs.

```python
# Python — dangerous! This hides bugs
try:
    data = load_file(path)
    value = data[index]          # IndexError is a BUG, not a file error!
    result = int(value)
except Exception:
    result = 0  # "Eh, just use a default"  ← BUG HIDDEN!
```

Rust makes this impossible by design. You handle `Result` with pattern matching. You
**cannot** catch a panic in the same way (well, `catch_unwind` exists, but it's
discouraged for normal control flow). The two error types live in completely separate
worlds:

```rust
use std::fs;

fn load_value(path: &str, index: usize) -> Result<i32, String> {
    // Recoverable: file might not exist → Result handles it
    let data = fs::read_to_string(path)
        .map_err(|e| format!("Failed to read file: {}", e))?;

    let lines: Vec<&str> = data.lines().collect();

    // Unrecoverable: if index is out of bounds, that's a bug.
    // We could also use .get(index) to make it recoverable — your choice!
    let value = lines[index];  // panics if bug

    value.trim().parse::<i32>()
        .map_err(|e| format!("Parse error: {}", e))
}
```

---

## Why No Exceptions?

If you're coming from Java, Python, C#, or C++, you probably expect a language to have
**exceptions** — a mechanism where errors are "thrown" up the call stack until someone
"catches" them. Rust deliberately does **not** have exceptions, and this is one of its
most important design decisions.

### What Are Exceptions?

In exception-based languages, any function can "throw" an error at any time:

```java
// Java
public String readFile(String path) throws IOException {
    // If something goes wrong, an IOException is THROWN
    // It flies up the call stack until someone catches it
    return new String(Files.readAllBytes(Paths.get(path)));
}

public void process() {
    try {
        String content = readFile("data.txt");
        // ... do stuff ...
    } catch (IOException e) {
        System.out.println("File error: " + e.getMessage());
    }
}
```

This looks reasonable. So what's the problem?

### Problem 1: Invisible Control Flow

The biggest problem with exceptions is that they create **invisible exit points** in your
code. Any function call can secretly throw an exception, and you can't tell by looking at
the call site:

```java
// Java — How many things can throw here?
public void processOrder(Order order) {
    // Can validate() throw? Who knows without reading its source code
    order.validate();

    // Can calculateTotal() throw? The signature doesn't say
    double total = order.calculateTotal();

    // Can charge() throw? Probably, but which exceptions?
    payment.charge(total);

    // Can sendConfirmation() throw? Maybe...
    email.sendConfirmation(order);
}
```

Every single method call is a potential hidden `goto` that jumps to some unknown `catch`
block somewhere up the call stack. The code above has **at least 4 invisible exit points**.
If `calculateTotal()` throws, the charge never happens — but does the order get rolled
back? What state is the system in? It's incredibly hard to reason about.

```
Exception-Based Control Flow (Invisible):

processOrder()
    │
    ├─→ validate()         ──→ throws! ──→ ???
    ├─→ calculateTotal()   ──→ throws! ──→ ???
    ├─→ charge()           ──→ throws! ──→ ???
    └─→ sendConfirmation() ──→ throws! ──→ ???
                                    ↓
                          Some catch block
                          somewhere up the
                          stack (maybe)

Rust's Control Flow (Explicit):

process_order()
    │
    ├─→ validate()         ──→ returns Result ──→ you handle it HERE
    ├─→ calculate_total()  ──→ returns Result ──→ you handle it HERE
    ├─→ charge()           ──→ returns Result ──→ you handle it HERE
    └─→ send_confirmation()──→ returns Result ──→ you handle it HERE
```

In Rust, every potential failure is visible in the function's return type. No surprises.
No invisible jumps.

### Problem 2: Performance Overhead

Exceptions are expensive. When an exception is thrown, the runtime must:

1. **Allocate** the exception object on the heap
2. **Capture** the stack trace (walking every frame on the call stack)
3. **Unwind** the stack, running destructors for every frame
4. **Search** for a matching catch handler (possibly in a completely different function)

This "zero-cost when not thrown, very expensive when thrown" model means:

- In C++, the compiler must generate extra metadata tables for unwinding (bloating binary size)
- In Java, creating an exception allocates memory and captures a full stack trace
- In Python, exception handling is part of normal control flow (iterators use `StopIteration`!),
  making it even harder to optimize

Rust avoids all of this. `Result<T, E>` is just a normal enum — no heap allocation, no
stack trace, no unwinding, no searching for handlers. Returning a `Result` is as cheap as
returning any other value.

```
Performance Comparison (approximate):

Operation                    | Cost
-----------------------------|---------------------------
Return Ok(value)             | ~1 nanosecond (same as any return)
Return Err(error)            | ~1 nanosecond (same as any return)
Throw exception (Java)       | ~1-10 microseconds (1000-10000x slower)
Throw exception (C++)        | ~5-50 microseconds (depends on depth)
Throw exception (Python)     | ~1-5 microseconds
```

### Problem 3: The "Catch Everything" Anti-Pattern

Because exceptions are invisible, developers often write overly broad catch blocks:

```python
# Python — The "Pokémon catch" (gotta catch 'em all!)
try:
    data = json.loads(response.text)
    user = data["users"][0]
    process_user(user)
except Exception:  # Catches EVERYTHING — even bugs!
    logger.error("Something went wrong")
    return None
```

This catches `KeyError` (a bug — the data format changed), `IndexError` (a bug — empty
list), `TypeError` (a bug — wrong argument types), and the actual `JSONDecodeError`
(a legitimate error). Bugs get silently swallowed, and debugging becomes a nightmare.

### Problem 4: Checked vs Unchecked — Java's Dilemma

Java tried to solve the invisible exception problem with **checked exceptions**: if a
method can throw a checked exception, it must declare it in its signature:

```java
// Java — Checked exception: MUST be declared
public void readFile() throws IOException {
    // ...
}

// Callers MUST either catch it or re-declare it
public void process() throws IOException {
    readFile();  // Must handle IOException somehow
}
```

Sounds good in theory! But in practice:

1. **Every caller in the chain** must either catch or re-declare the exception, leading to
   `throws` clauses that grow endlessly
2. **Changing an implementation detail** (e.g., switching from file I/O to network I/O) changes
   the public API — a new throws clause is a breaking change
3. **Developers revolt** and wrap everything in `RuntimeException` to escape the system:

```java
// The "I give up" pattern — defeats the entire purpose
try {
    riskyOperation();
} catch (CheckedException e) {
    throw new RuntimeException(e);  // Convert to unchecked — compiler shuts up
}
```

Java 8's lambda expressions made this worse: functional interfaces like `Function<T, R>`
don't declare checked exceptions, so you can't easily use methods that throw inside
lambdas.

Rust looked at this 20-year experiment in Java and said: "What if errors were just normal
return values? Then no special 'throws' clause is needed — the return type IS the error
declaration."

---

## The Exception Problem in Other Languages

Let's look at how different languages handle errors, and what problems each approach creates.
This will help you appreciate Rust's design.

### Java: The Checked Exception Experiment

Java (1995) introduced a bold idea: force developers to handle errors by making them part
of the method signature. This created a **two-tier system**:

```java
// CHECKED exceptions — compiler forces you to handle them
public class FileProcessor {
    // Must declare IOException in the signature
    public String readConfig() throws IOException {
        return new String(Files.readAllBytes(Paths.get("config.txt")));
    }

    // Every caller inherits the obligation
    public void initialize() throws IOException {
        String config = readConfig();  // Must handle or propagate
        // ...
    }
}

// UNCHECKED exceptions — compiler ignores them
public class Calculator {
    public int divide(int a, int b) {
        return a / b;  // Can throw ArithmeticException — but no declaration needed!
    }
}
```

**The hierarchy bloat problem:**

```
Java Exception Hierarchy (simplified)
─────────────────────────────────────
Throwable
├── Error (unchecked — JVM failures, OutOfMemoryError)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── ...150+ more...
└── Exception
    ├── RuntimeException (unchecked — bugs)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   └── ...200+ more...
    └── <everything else> (checked — must handle)
        ├── IOException
        │   ├── FileNotFoundException
        │   ├── SocketException
        │   │   ├── ConnectException
        │   │   └── BindException
        │   └── ...50+ more...
        ├── SQLException
        └── ...hundreds more...
```

Real-world Java codebases end up with massive exception hierarchies, and developers just
catch the broad base class to avoid dealing with all of them.

### Python: Duck-Typed Chaos

Python uses exceptions for *everything* — even normal control flow:

```python
# Python — Exceptions are NORMAL control flow
for item in my_list:
    print(item)
# Under the hood, the for loop catches StopIteration to know when to stop!

# What exceptions can this function raise?
def get_user_data(user_id):
    response = requests.get(f"/api/users/{user_id}")  # ConnectionError? Timeout?
    data = response.json()                              # JSONDecodeError?
    return data["name"]                                 # KeyError?

# Answer: Nobody knows without reading EVERY line of source code,
# including the source of every library you call!
```

Python has no mechanism to declare what exceptions a function might raise (type hints
don't cover exceptions). The result: **every function call is a mystery**.

```python
# The only "safe" Python code is covered in try/except
def safe_get_user(user_id):
    try:
        return get_user_data(user_id)
    except requests.ConnectionError:
        return None    # Network error
    except requests.Timeout:
        return None    # Slow server
    except KeyError:
        return None    # Malformed data — but also hides typos in YOUR code!
    except Exception:
        return None    # "Just in case" — hides ALL bugs
```

### C++: Exception Safety Is Nearly Impossible

C++ (1990) added exceptions, then spent 30 years trying to use them correctly. C++ defines
three levels of **exception safety guarantees**:

```cpp
// C++ exception safety levels:

// 1. BASIC guarantee: No resource leaks, objects remain valid (but maybe changed)
void basic_safe(std::vector<int>& v) {
    v.push_back(42);  // If allocation throws, v is still valid (but unchanged)
}

// 2. STRONG guarantee: Operation succeeds completely or has no effect (rollback)
void strong_safe(std::vector<int>& v) {
    auto copy = v;        // Copy first
    copy.push_back(42);   // Modify the copy
    std::swap(v, copy);   // Swap only if everything succeeded (noexcept)
}

// 3. NOTHROW guarantee: Never throws — the gold standard
int safe_add(int a, int b) noexcept {
    return a + b;  // Simple operations that can't fail
}
```

The problem? Writing strongly exception-safe C++ code is **agonizingly difficult**:

```cpp
// C++ — How many things can throw here?
class Database {
public:
    void transfer(Account& from, Account& to, double amount) {
        from.withdraw(amount);    // If this throws, both accounts unchanged ✓
        to.deposit(amount);       // If THIS throws, money disappeared from `from`
                                  // but never arrived at `to`! 💀
    }
};
// Getting this right requires careful "copy and swap" or transaction patterns
```

Many major C++ projects (including Google's style guide) **ban exceptions entirely** and use
error codes or `std::expected` (C++23) instead — essentially reinventing Rust's `Result`.

### Go: Close to Rust, But With a Gap

Go (2009) took a similar approach to Rust — errors are values, not exceptions:

```go
// Go — errors are return values
func readFile(path string) (string, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return "", fmt.Errorf("reading %s: %w", path, err)
    }
    return string(data), nil
}

// Caller MUST deal with the error (in theory)
func main() {
    content, err := readFile("config.toml")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(content)
}
```

Go's approach is spiritually similar to Rust's, but has a critical weakness: **the
compiler doesn't enforce error handling**:

```go
// Go — You can silently ignore errors!
func main() {
    content, _ := readFile("config.toml")  // Oops, error ignored with _
    // OR even worse:
    content, err := readFile("config.toml")
    // `err` is never checked — Go compiler is fine with this!
    fmt.Println(content)  // Might be empty string, no warning
}
```

In Rust, the compiler warns you if you don't use a `Result`:

```rust
fn main() {
    // ⚠️ warning: unused `Result` that must be used
    std::fs::read_to_string("config.toml");
    // The compiler literally says: "this Result may be an Err that should be handled"
}
```

### Cross-Language Comparison Table

```
Language  │ Mechanism              │ Enforced? │ Visible? │ Performance │ Composable?
──────────┼────────────────────────┼───────────┼──────────┼─────────────┼────────────
C         │ Return codes (int)     │ No        │ Somewhat │ Fast        │ No
C++       │ Exceptions             │ No        │ No       │ Costly      │ Somewhat
Java      │ Checked + Unchecked    │ Partially │ Partially│ Costly      │ No
Python    │ Exceptions             │ No        │ No       │ Costly      │ No
Go        │ Multiple returns       │ No        │ Yes      │ Fast        │ Somewhat
Haskell   │ Either / Maybe         │ Yes       │ Yes      │ Fast        │ Yes
Rust      │ Result<T, E> / panic!  │ Yes       │ Yes      │ Fast        │ Yes
```

---

## Rust's Approach: Errors as Values

Now that we've seen what other languages do wrong (or partly right), let's see Rust's
solution in detail.

### The Core Idea

In Rust, a function that can fail returns a `Result<T, E>`:

```rust
enum Result<T, E> {
    Ok(T),    // Success — contains the value
    Err(E),   // Failure — contains the error
}
```

That's it. `Result` is just an enum — the same kind of enum you've been using since
Stage 4. No special language feature. No runtime support. No magic.

```rust
use std::num::ParseIntError;

// This function can fail — the return type makes it OBVIOUS
fn parse_age(input: &str) -> Result<u8, ParseIntError> {
    input.trim().parse::<u8>()
}

fn main() {
    let inputs = vec!["25", "abc", "300", "42"];

    for input in inputs {
        match parse_age(input) {
            Ok(age) => println!("  '{}' → age {}", input, age),
            Err(e) => println!("  '{}' → error: {}", input, e),
        }
    }
}
// Output:
//   '25' → age 25
//   'abc' → error: invalid digit found in string
//   '300' → error: number too large to fit in target type
//   '42' → age 42
```

### Why Values Are Better Than Exceptions

**1. Errors are visible in the function signature:**

```rust
// You can tell EXACTLY what can fail just by reading the type:
fn connect(addr: &str) -> Result<TcpStream, io::Error>
fn parse(json: &str) -> Result<Config, serde_json::Error>
fn divide(a: f64, b: f64) -> Result<f64, MathError>

// And what CANNOT fail:
fn add(a: i32, b: i32) -> i32  // No Result = no failure possible
fn len(s: &str) -> usize       // Always succeeds
```

**2. The compiler forces you to handle errors:**

```rust
fn main() {
    // ❌ This won't compile (if you try to use the value)
    let file: String = std::fs::read_to_string("data.txt");
    //                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // Error: expected `String`, found `Result<String, io::Error>`

    // ✅ You MUST acknowledge the Result
    let file: String = std::fs::read_to_string("data.txt")
        .expect("Failed to read data.txt");
}
```

**3. Errors compose with standard combinators:**

```rust
use std::fs;

// Chain multiple fallible operations cleanly
fn load_config() -> Result<Config, Box<dyn std::error::Error>> {
    let text = fs::read_to_string("config.toml")?;    // ? propagates errors
    let config: Config = toml::from_str(&text)?;       // ? propagates errors
    Ok(config)
}
```

**4. No performance penalty:**

```rust
// Result<T, E> is stored inline — no heap allocation
// On x86-64, Result<u64, io::Error> is just 16 bytes on the stack:
//
// ┌──────────┬──────────────────────┐
// │ tag (8B) │ payload (8B)         │
// │ 0 = Ok   │ the u64 value        │
// │ 1 = Err  │ io::Error details    │
// └──────────┴──────────────────────┘
//
// No heap. No stack trace. No unwinding tables.
```

### The `#[must_use]` Secret

You might wonder: "What if I just call a function that returns `Result` and don't
use the return value?" Rust has you covered:

```rust
fn save_data(data: &str) -> Result<(), std::io::Error> {
    std::fs::write("output.txt", data)
}

fn main() {
    save_data("hello");
    // ⚠️ warning: unused `Result` that must be used
    // ⚠️ note: this `Result` may be an `Err` variant, which should be handled
}
```

`Result` is marked with `#[must_use]`, so the compiler **warns** you if you ignore it.
This is something Go doesn't do — in Go, you can silently discard errors with no warning.

---

## The Error Trait

Rust provides a standard trait for error types: `std::error::Error`. This trait gives
errors a consistent interface and enables error chaining.

### The Definition

```rust
// Simplified from the standard library
pub trait Error: Display + Debug {
    // The underlying cause of this error (if any)
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None  // Default: no underlying cause
    }
}
```

Every error type in Rust should implement three things:

1. **`Display`** — A user-friendly error message (`"file not found: config.toml"`)
2. **`Debug`** — A developer-friendly representation (`FileNotFound("config.toml")`)
3. **`source()`** — An optional link to the underlying cause (for error chains)

### Implementing the Error Trait

```rust
use std::fmt;
use std::error::Error;

// Step 1: Define your error type
#[derive(Debug)]
enum AppError {
    NotFound(String),
    PermissionDenied(String),
    ParseFailed { input: String, reason: String },
}

// Step 2: Implement Display (user-facing message)
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(path) =>
                write!(f, "not found: {}", path),
            AppError::PermissionDenied(path) =>
                write!(f, "permission denied: {}", path),
            AppError::ParseFailed { input, reason } =>
                write!(f, "failed to parse '{}': {}", input, reason),
        }
    }
}

// Step 3: Implement Error (enables error chaining and downcasting)
impl Error for AppError {}

fn load_config(path: &str) -> Result<String, AppError> {
    if path.is_empty() {
        return Err(AppError::NotFound(path.to_string()));
    }
    // ... in real code, actually read the file
    Ok(String::from("key = value"))
}

fn main() {
    match load_config("") {
        Ok(config) => println!("Loaded: {}", config),
        Err(e) => {
            println!("Error: {}", e);        // Uses Display
            println!("Debug: {:?}", e);      // Uses Debug
        }
    }
}
// Output:
//   Error: not found:
//   Debug: NotFound("")
```

### Error Chaining with `source()`

When one error causes another, you can link them together:

```rust
use std::fmt;
use std::error::Error;
use std::num::ParseIntError;

#[derive(Debug)]
struct ConfigError {
    message: String,
    source: Option<Box<dyn Error>>,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "config error: {}", self.message)
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        self.source.as_ref().map(|e| e.as_ref() as &(dyn Error + 'static))
    }
}

fn parse_port(s: &str) -> Result<u16, ConfigError> {
    s.parse::<u16>().map_err(|e| ConfigError {
        message: format!("invalid port: '{}'", s),
        source: Some(Box::new(e)),
    })
}

fn main() {
    if let Err(e) = parse_port("banana") {
        println!("Error: {}", e);

        // Walk the error chain
        let mut source = e.source();
        while let Some(cause) = source {
            println!("  Caused by: {}", cause);
            source = cause.source();
        }
    }
}
// Output:
//   Error: config error: invalid port: 'banana'
//     Caused by: invalid digit found in string
```

This error chaining is the foundation for crates like `anyhow` and `thiserror` that
we'll explore in later tutorials.

### Why a Trait Instead of a Base Class?

In Java, all exceptions extend `Exception` (or `Throwable`). This creates a rigid class
hierarchy. In Rust, `Error` is a *trait* — any type can implement it without inheriting
from anything:

```
Java: Rigid Hierarchy                  Rust: Flexible Traits
─────────────────────                  ────────────────────
  Throwable                            Any type can implement Error:
  └── Exception                         • enum AppError: Error
      └── IOException                   • struct IoError: Error
          └── FileNotFoundException     • struct ParseError: Error
              └── ...deep nesting...    • (no hierarchy needed!)
```

---

## A Mental Model: When to Panic vs Result

This is one of the most important decisions you'll make in Rust. Here's a clear framework:

### Use `panic!` When...

| Situation | Why Panic? | Example |
|-----------|-----------|---------|
| **A bug in your code** | If this happens, the code is WRONG | Index out of bounds |
| **Violated invariants** | The impossible happened | A sorted vec isn't sorted |
| **Unrecoverable state** | No way to continue usefully | Critical config missing at startup |
| **Prototyping/examples** | You'll add proper errors later | `unwrap()` for quick iteration |
| **Tests** | Tests SHOULD panic on failure | `assert_eq!`, `unwrap()` in tests |

```rust
// GOOD uses of panic

// 1. Bug detection — this should never happen
fn get_first(v: &[i32]) -> i32 {
    assert!(!v.is_empty(), "get_first called on empty slice — this is a bug!");
    v[0]
}

// 2. Violated invariants
fn process_sorted(data: &[i32]) {
    debug_assert!(data.windows(2).all(|w| w[0] <= w[1]),
        "Data must be sorted — contract violation!");
    // ... process sorted data ...
}

// 3. Prototyping — "I'll handle this properly later"
fn quick_prototype() {
    let config = std::fs::read_to_string("config.toml").unwrap();
    let port: u16 = config.lines()
        .find(|l| l.starts_with("port"))
        .unwrap()
        .split('=')
        .nth(1)
        .unwrap()
        .trim()
        .parse()
        .unwrap();
    println!("Port: {}", port);
}
```

### Use `Result` When...

| Situation | Why Result? | Example |
|-----------|------------|---------|
| **Expected failures** | The caller should decide what to do | File not found |
| **User input** | Users WILL give bad input | Parsing, validation |
| **I/O operations** | Networks and disks are unreliable | HTTP requests, file reads |
| **External systems** | You can't control them | Databases, APIs |
| **Library functions** | Let the caller choose the strategy | Any public API |

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

// GOOD uses of Result

// 1. File I/O — files might not exist
fn read_config(path: &str) -> Result<String, io::Error> {
    fs::read_to_string(path)
}

// 2. User input — users type weird stuff
fn parse_user_age(input: &str) -> Result<u8, String> {
    let age: u8 = input.trim().parse()
        .map_err(|e: ParseIntError| format!("Invalid age '{}': {}", input, e))?;
    if age > 150 {
        return Err(format!("Age {} is unrealistic", age));
    }
    Ok(age)
}

// 3. Network operations — the cloud is someone else's computer
fn fetch_data(url: &str) -> Result<String, Box<dyn std::error::Error>> {
    // In real code: reqwest::get(url)?.text()?
    Ok(String::from("data"))
}
```

### The Decision Flowchart

```
                      An error occurred!
                            │
                            ▼
                 Is this a BUG in your code?
                   /                    \
                 YES                     NO
                  │                       │
                  ▼                       ▼
              panic!()            Can the caller recover?
                                   /               \
                                 YES                 NO
                                  │                   │
                                  ▼                   ▼
                            Result<T, E>       panic!() or
                                              exit with message
                  
Examples:                           Examples:
• assert!() failures                • File not found
• unwrap() on None                  • Parse errors
• Index out of bounds               • Network timeouts
• "impossible" states               • Permission denied
```

### A Practical Rule of Thumb

> **If the error could happen in production with valid code, use `Result`.**
> **If the error means the code itself is wrong, use `panic!`.**

Think of it this way: if you deploy perfect, bug-free code (theoretically!), would this
error still happen? File not found: yes (user might delete the file). Network timeout: yes
(the internet is unreliable). Index out of bounds: NO — that means your logic is wrong.

---

## Historical Context: The Evolution of Error Handling

Error handling has evolved dramatically over ~50 years of programming language design.
Understanding this evolution helps you see why Rust's approach is the way it is.

### Era 1: Return Codes (1970s — C)

The original approach: functions return a special value to indicate failure.

```c
// C — return codes
#include <stdio.h>
#include <errno.h>

int main() {
    FILE *f = fopen("data.txt", "r");  // Returns NULL on failure

    if (f == NULL) {
        // Check the global `errno` variable for details
        printf("Error %d: %s\n", errno, strerror(errno));
        return 1;
    }

    // ... use f ...
    fclose(f);
    return 0;
}
```

**Problems:**
- **Easy to ignore**: Nothing forces you to check the return value
- **Conflates error with value**: How do you distinguish `-1` as an error from `-1` as a
  valid result? (Answer: you can't always)
- **Global state**: `errno` is a global variable — not thread-safe in early C
- **No type safety**: Errors are just `int` — you need a manual to know what `-1` means

```
C Error Handling:

    fopen("data.txt", "r")
         │
         ├── Returns FILE* on success
         └── Returns NULL on failure 
             └── Check errno (global int) for details
                  └── 2 = ENOENT (no such file)
                  └── 13 = EACCES (permission denied)
                  └── ... dozens of errno values ...

    Problem: NOTHING forces you to check NULL!
    
    FILE *f = fopen("data.txt", "r");
    fread(buf, 1, 100, f);  // If f is NULL → segfault 💥 
```

### Era 2: Exceptions (1990s — C++, Java)

C++ (1990) and Java (1995) introduced exceptions to solve C's problems:

```
The Promise: "Separate error handling from normal code!"

Normal Code Path:
    try {
        open_file()         // On success:
        read_data()         //   runs top to bottom
        process_data()      //   clean, readable
        save_results()
    }

Error Handling Path:
    catch (FileError e) {   // All error handling
        handle_error()      //   is collected HERE
    }
```

**What went right:**
- Errors can't be accidentally confused with return values
- Stack unwinding cleans up resources (in theory)
- Errors propagate automatically — no manual checking at every call

**What went wrong:**
- Invisible control flow (covered in detail above)
- Performance overhead for throwing
- Exception safety is incredibly hard (C++)
- Checked exceptions become a burden (Java)
- Exceptions used for non-error control flow (Python's `StopIteration`)

### Era 3: Algebraic Types (2000s — Haskell, OCaml, Scala)

Functional languages realized: what if errors were just normal values of a sum type?

```haskell
-- Haskell's Either type (1990, but popularized in 2000s)
data Either a b = Left a | Right b

-- Convention: Left = error, Right = success
readFile :: FilePath -> IO (Either IOError String)

-- You MUST pattern match to get the value
case readFile "data.txt" of
    Left err     -> putStrLn ("Error: " ++ show err)
    Right content -> putStrLn content
```

```ocaml
(* OCaml's result type *)
type ('a, 'b) result = Ok of 'a | Error of 'b

let parse_int s =
  match int_of_string_opt s with
  | Some n -> Ok n
  | None -> Error ("Not a number: " ^ s)
```

**What went right:**
- Errors are visible in the type signature
- The compiler forces you to handle both cases
- Composable with `map`, `bind` (`and_then`), and other combinators
- No runtime overhead — it's just a tagged union

**What went wrong:**
- Error handling can become verbose without syntactic sugar
- Nested `Either` / `Result` types get unwieldy
- Monadic composition has a learning curve

### Era 4: Rust's Synthesis (2015)

Rust took the best ideas from every era:

```
From C:          Errors are cheap values (no runtime overhead)
From Java:       The compiler helps enforce error handling (#[must_use])
From Haskell:    Algebraic types (Result<T, E> is functionally Either<E, T>)
From Go:         Multiple return concept (but type-safe, not just convention)
From experience: The ? operator (syntactic sugar for error propagation)
```

```rust
// Rust's synthesis: the best of all worlds
use std::fs;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    Io(io::Error),
    Parse(ParseIntError),
    Missing(String),
}

impl From<io::Error> for ConfigError {
    fn from(e: io::Error) -> Self { ConfigError::Io(e) }
}

impl From<ParseIntError> for ConfigError {
    fn from(e: ParseIntError) -> Self { ConfigError::Parse(e) }
}

impl std::fmt::Display for ConfigError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConfigError::Io(e) => write!(f, "I/O error: {}", e),
            ConfigError::Parse(e) => write!(f, "parse error: {}", e),
            ConfigError::Missing(key) => write!(f, "missing key: {}", key),
        }
    }
}

impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::Io(e) => Some(e),
            ConfigError::Parse(e) => Some(e),
            ConfigError::Missing(_) => None,
        }
    }
}

fn load_port(path: &str) -> Result<u16, ConfigError> {
    let content = fs::read_to_string(path)?;  // io::Error → ConfigError via From
    let port_str = content.lines()
        .find(|l| l.starts_with("port"))
        .ok_or_else(|| ConfigError::Missing("port".to_string()))?;
    let port: u16 = port_str
        .split('=')
        .nth(1)
        .ok_or_else(|| ConfigError::Missing("port value".to_string()))?
        .trim()
        .parse()?;  // ParseIntError → ConfigError via From
    Ok(port)
}

fn main() {
    match load_port("server.toml") {
        Ok(port) => println!("Server starting on port {}", port),
        Err(e) => {
            eprintln!("Failed to load config: {}", e);
            if let Some(source) = std::error::Error::source(&e) {
                eprintln!("  Caused by: {}", source);
            }
            std::process::exit(1);
        }
    }
}
```

### Timeline Visualization

```
Evolution of Error Handling
═══════════════════════════════════════════════════════════════════════

1970s          1990          1995         2009      2010s       2015
  │              │             │            │         │           │
  C            C++           Java          Go     Haskell*     Rust
  │              │             │            │      (popular.)    │
  ▼              ▼             ▼            ▼         ▼          ▼

Return         Exceptions   Checked      Multiple  Either/     Result<T,E>
Codes          (throw/      Exceptions   Returns   Maybe       + panic!
(int, NULL,    catch)       + Unchecked  (val,err) (algebraic  + ? operator
errno)                                             types)      + #[must_use]

Problems:      Problems:    Problems:    Problems: Problems:    ✓ Type-safe
• Ignorable    • Invisible  • Verbose    • Ignor-  • Verbose   ✓ Enforced
• Ambiguous      control    • Hierarchy    able    • Learning  ✓ Composable
• No types       flow         bloat      • No type  curve     ✓ Zero-cost
               • Costly     • Dev revolt   enforce           ✓ Visible
               • Hard to    • Lambdas                        ✓ Ergonomic (?)
                 reason       conflict

* Haskell existed since 1990 but Either/Maybe patterns gained mainstream
  attention in the 2010s through Scala, Swift, and Rust.
```

---

## Exercises

### Exercise 1: Classify the Errors

For each scenario below, decide: should this use `panic!` or `Result<T, E>`? Write your
reasoning.

```rust
// Scenario A: A function that reads a temperature sensor
fn read_temperature() -> /* panic! or Result? */ {
    // The sensor might be disconnected
    // The reading might be out of the valid range
    todo!()
}

// Scenario B: A function that accesses a HashMap that was just populated
fn get_known_key(map: &HashMap<String, i32>) -> i32 {
    // We JUST inserted "total" into the map one line above
    // It should definitely be there
    todo!()
}

// Scenario C: A web server handler for user login
fn handle_login(username: &str, password: &str) -> /* panic! or Result? */ {
    // Username might not exist in database
    // Password might be wrong
    // Database connection might fail
    todo!()
}
```

<details>
<summary>💡 Solution</summary>

```rust
use std::collections::HashMap;

// Scenario A: Result — sensor disconnection is an expected real-world failure
fn read_temperature() -> Result<f64, String> {
    // The sensor is a hardware device — it WILL fail sometimes.
    // This is a recoverable error. The caller might retry,
    // use a cached value, or alert the operator.
    let raw_value: f64 = 23.5; // simulate reading

    if raw_value < -273.15 || raw_value > 1000.0 {
        return Err(format!("Temperature {} out of valid range", raw_value));
    }
    Ok(raw_value)
}

// Scenario B: panic! (via unwrap/expect or indexing) — if the key isn't
// there, our code is WRONG. This is a bug, not an expected failure.
fn get_known_key(map: &HashMap<String, i32>) -> i32 {
    // We KNOW "total" is in the map — we just put it there.
    // If it's missing, something is fundamentally broken.
    *map.get("total").expect("BUG: 'total' key should always exist")
    // Alternative: map["total"] — panics with a less descriptive message
}

// Scenario C: Result — ALL of these failures are normal in production
fn handle_login(username: &str, password: &str) -> Result<String, String> {
    // Users mistype passwords every day. That's not a bug.
    // Databases go down. That's not a bug.
    // None of these warrant crashing the entire server!
    if username == "admin" && password == "secret" {
        Ok(String::from("token-abc123"))
    } else {
        Err(String::from("Invalid credentials"))
    }
}

fn main() {
    // A: Result
    match read_temperature() {
        Ok(temp) => println!("Temperature: {:.1}°C", temp),
        Err(e) => println!("Sensor error: {}", e),
    }

    // B: panic! (if bug)
    let mut map = HashMap::new();
    map.insert("total".to_string(), 42);
    println!("Total: {}", get_known_key(&map));

    // C: Result
    match handle_login("admin", "wrong") {
        Ok(token) => println!("Logged in! Token: {}", token),
        Err(e) => println!("Login failed: {}", e),
    }
}
```
</details>

### Exercise 2: Convert Exceptions to Results

Take this pseudocode (written in exception-style) and rewrite it as idiomatic Rust using
`Result`:

```
// Pseudocode (exception-style)
function loadUserProfile(userId):
    try:
        response = httpGet("/api/users/" + userId)     // might throw NetworkError
        data = parseJson(response)                      // might throw ParseError
        return data.name                                // might throw KeyError
    catch NetworkError:
        return "Unknown User"
    catch ParseError:
        log("Malformed response for user " + userId)
        throw                                           // re-throw
    catch KeyError:
        return "Anonymous"
```

<details>
<summary>💡 Solution</summary>

```rust
use std::fmt;
use std::collections::HashMap;

#[derive(Debug)]
enum ProfileError {
    Network(String),
    Parse(String),
    MissingField(String),
}

impl fmt::Display for ProfileError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ProfileError::Network(msg) => write!(f, "network error: {}", msg),
            ProfileError::Parse(msg) => write!(f, "parse error: {}", msg),
            ProfileError::MissingField(key) => write!(f, "missing field: {}", key),
        }
    }
}

// Simulate an HTTP GET (in real code, use reqwest or similar)
fn http_get(url: &str) -> Result<String, ProfileError> {
    if url.contains("404") {
        Err(ProfileError::Network(format!("404 Not Found: {}", url)))
    } else {
        Ok(r#"{"name": "Alice", "age": 30}"#.to_string())
    }
}

// Simulate JSON parsing
fn parse_json(raw: &str) -> Result<HashMap<String, String>, ProfileError> {
    // In real code, use serde_json
    if raw.starts_with('{') {
        let mut map = HashMap::new();
        map.insert("name".to_string(), "Alice".to_string());
        Ok(map)
    } else {
        Err(ProfileError::Parse(format!("invalid JSON: {}", raw)))
    }
}

fn load_user_profile(user_id: &str) -> Result<String, ProfileError> {
    // Network error → we choose to recover with a default
    let response = match http_get(&format!("/api/users/{}", user_id)) {
        Ok(resp) => resp,
        Err(ProfileError::Network(_)) => return Ok("Unknown User".to_string()),
        Err(e) => return Err(e),
    };

    // Parse error → log AND propagate (re-throw equivalent)
    let data = match parse_json(&response) {
        Ok(d) => d,
        Err(e) => {
            eprintln!("Malformed response for user {}: {}", user_id, e);
            return Err(e);  // Propagate — equivalent to re-throw
        }
    };

    // Missing field → recover with default
    match data.get("name") {
        Some(name) => Ok(name.clone()),
        None => Ok("Anonymous".to_string()),
    }
}

fn main() {
    // Normal case
    println!("{}", load_user_profile("alice").unwrap());
    // Network error case → returns "Unknown User" (recovered)
    println!("{}", load_user_profile("404").unwrap());
}
```

**Key insight:** In Rust, each error is handled **at the point it occurs**. There's no
distant `catch` block — you decide right here, right now, what to do with each failure.
This makes the control flow crystal clear.
</details>

### Exercise 3: Design an Error Type

Design an error enum for a simple calculator that supports `+`, `-`, `*`, `/` on integers.
Consider what could go wrong and implement `Display` for clear messages.

<details>
<summary>💡 Solution</summary>

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
enum CalcError {
    DivisionByZero,
    Overflow { operation: String, left: i64, right: i64 },
    InvalidOperator(String),
    ParseError { input: String, reason: String },
}

impl fmt::Display for CalcError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            CalcError::DivisionByZero =>
                write!(f, "division by zero"),
            CalcError::Overflow { operation, left, right } =>
                write!(f, "overflow: {} {} {} is too large", left, operation, right),
            CalcError::InvalidOperator(op) =>
                write!(f, "unknown operator: '{}'", op),
            CalcError::ParseError { input, reason } =>
                write!(f, "cannot parse '{}': {}", input, reason),
        }
    }
}

impl Error for CalcError {}

fn calculate(left: i64, op: &str, right: i64) -> Result<i64, CalcError> {
    match op {
        "+" => left.checked_add(right)
            .ok_or(CalcError::Overflow {
                operation: op.to_string(), left, right
            }),
        "-" => left.checked_sub(right)
            .ok_or(CalcError::Overflow {
                operation: op.to_string(), left, right
            }),
        "*" => left.checked_mul(right)
            .ok_or(CalcError::Overflow {
                operation: op.to_string(), left, right
            }),
        "/" => {
            if right == 0 {
                Err(CalcError::DivisionByZero)
            } else {
                Ok(left / right)
            }
        }
        other => Err(CalcError::InvalidOperator(other.to_string())),
    }
}

fn eval(expression: &str) -> Result<i64, CalcError> {
    let parts: Vec<&str> = expression.split_whitespace().collect();
    if parts.len() != 3 {
        return Err(CalcError::ParseError {
            input: expression.to_string(),
            reason: "expected format: <number> <op> <number>".to_string(),
        });
    }

    let left: i64 = parts[0].parse().map_err(|_| CalcError::ParseError {
        input: parts[0].to_string(),
        reason: "not a valid integer".to_string(),
    })?;

    let right: i64 = parts[2].parse().map_err(|_| CalcError::ParseError {
        input: parts[2].to_string(),
        reason: "not a valid integer".to_string(),
    })?;

    calculate(left, parts[1], right)
}

fn main() {
    let expressions = vec![
        "10 + 20",
        "100 / 0",
        "5 ^ 3",
        "abc + 1",
        "9223372036854775807 + 1",  // i64::MAX + 1
    ];

    for expr in expressions {
        match eval(expr) {
            Ok(result) => println!("  {} = {}", expr, result),
            Err(e) => println!("  {} → Error: {}", expr, e),
        }
    }
}
// Output:
//   10 + 20 = 30
//   100 / 0 → Error: division by zero
//   5 ^ 3 → Error: unknown operator: '^'
//   abc + 1 → Error: cannot parse 'abc': not a valid integer
//   9223372036854775807 + 1 → Error: overflow: 9223372036854775807 + 1 is too large
```
</details>

---

## Summary

| Concept | Detail |
|---------|--------|
| **Two error categories** | Recoverable (`Result<T, E>`) and unrecoverable (`panic!`) |
| **No exceptions** | Rust has no throw/catch — errors are values, not hidden control flow |
| **`Result<T, E>`** | An enum with `Ok(T)` and `Err(E)` — the standard way to handle recoverable errors |
| **`panic!`** | For bugs, violated invariants, and impossible states — stops the thread |
| **`#[must_use]`** | The compiler warns if you ignore a `Result` — unlike Go or C |
| **`Error` trait** | Standard interface: `Display` + `Debug` + optional `source()` for chaining |
| **When to panic** | Bugs, broken invariants, prototyping, tests |
| **When to use Result** | I/O, user input, network, parsing — anything that can *reasonably* fail |
| **History** | Return codes (C) → Exceptions (C++/Java) → Algebraic types (Haskell) → Rust's synthesis |
| **Errors as values** | Visible in signatures, enforced by compiler, zero-cost, composable |

---

**Next:** [panic! & Unrecoverable Errors →](./02-panic.md)

<p align="center"><i>Tutorial 1 of 7 — Stage 7: Error Handling</i></p>

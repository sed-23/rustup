# panic! and Unrecoverable Errors 🎯

> **"A panic is Rust's way of saying: this is a bug, not a bump in the road — we stop here."**

---

## Table of Contents

- [What Is a Panic?](#what-is-a-panic)
- [The panic! Macro](#the-panic-macro)
- [When Rust Panics Automatically](#when-rust-panics-automatically)
- [Unwinding vs Aborting](#unwinding-vs-aborting)
- [Stack Traces and RUST_BACKTRACE](#stack-traces-and-rust_backtrace)
- [Catching Panics with catch_unwind](#catching-panics-with-catch_unwind)
- [The Panic Hook](#the-panic-hook)
- [Panics Across Languages](#panics-across-languages)
- [Panic in Multi-threaded Code](#panic-in-multi-threaded-code)
- [When to Use panic! (Guidelines)](#when-to-use-panic-guidelines)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is a Panic?

Rust splits all errors into two categories:

| Category | Meaning | Mechanism |
|---|---|---|
| **Recoverable** | Expected thing went wrong (file missing, bad input) | `Result<T, E>` |
| **Unrecoverable** | A *bug* — something that should never happen | `panic!` |

A **panic** is Rust saying:

> "Something went so catastrophically wrong that we **cannot** continue safely.
> This is not a hiccup — this is a **bug** in the program."

Think of it like a fire alarm in a building. You don't negotiate with a fire alarm — you evacuate. A panic is the program's fire alarm.

```
    Normal execution
    ─────────────────────────────────────►

    ┌───────────────────────────────────┐
    │  Your code is running happily...  │
    │                                   │
    │  fn process(index: usize) {       │
    │      let v = vec![1, 2, 3];       │
    │      let val = v[index]; ◄──────────── index = 99 ?!
    │  }                                │
    └───────────┬───────────────────────┘
                │
                ▼  PANIC!
    ┌───────────────────────────────────┐
    │  "index out of bounds: the len    │
    │   is 3 but the index is 99"       │
    │                                   │
    │  Thread panicked ──► unwind stack │
    │  ──► run destructors ──► exit     │
    └───────────────────────────────────┘
```

### The Key Distinction

This is foundational to Rust's error philosophy:

- **File not found** → recoverable → use `Result`
- **Index out of bounds** → bug → `panic!`
- **Network timeout** → recoverable → use `Result`
- **Called unwrap() on None** → bug → `panic!`

A panic means the **programmer** made a mistake, not the user or the environment.

---

## The panic! Macro

The simplest way to trigger a panic is with the `panic!` macro.

### Basic Usage

```rust
fn main() {
    panic!("something went terribly wrong");
}
```

Output:
```
thread 'main' panicked at 'something went terribly wrong', src/main.rs:2:5
```

The message tells you:
1. **Which thread** panicked (`main`)
2. **The message** you provided
3. **Where** it happened (file, line, column)

### Panic with Formatted Messages

`panic!` supports the same formatting as `println!`:

```rust
fn main() {
    let user_id = 42;
    let max_id = 30;

    panic!(
        "user ID {} exceeds maximum allowed ID of {}",
        user_id, max_id
    );
}
```

Output:
```
thread 'main' panicked at 'user ID 42 exceeds maximum allowed ID of 30', src/main.rs:4:5
```

### Panic with No Message

You can call `panic!()` with no arguments, but please don't — always give a message:

```rust
// Bad — unhelpful
panic!();

// Good — explains what went wrong
panic!("connection pool exhausted: this should never happen with proper config");
```

### Using todo!(), unimplemented!(), and unreachable!()

Rust provides specialized panic macros for common situations:

```rust
fn calculate_tax(income: f64) -> f64 {
    todo!("implement tax calculation later")
    // Panics with: "not yet implemented: implement tax calculation later"
}

fn exotic_sort(data: &mut [i32]) {
    unimplemented!("this algorithm is not available on this platform")
    // Panics with: "not implemented: this algorithm is not..."
}

fn direction_from_code(code: u8) -> &'static str {
    match code {
        0 => "North",
        1 => "South",
        2 => "East",
        3 => "West",
        _ => unreachable!("direction code must be 0-3, got {}", code),
        // Panics with: "internal error: entered unreachable code: ..."
    }
}
```

| Macro | When to Use |
|---|---|
| `panic!("msg")` | General-purpose "this is a bug" |
| `todo!()` | Placeholder — you plan to write this code later |
| `unimplemented!()` | This code path intentionally has no implementation |
| `unreachable!()` | Logically impossible to reach this point |

All four of these macros cause a panic. They differ only in intent and message formatting.

---

## When Rust Panics Automatically

You don't always call `panic!` yourself. Rust triggers panics automatically in several situations:

### 1. Index Out of Bounds

```rust
fn main() {
    let numbers = vec![10, 20, 30];
    let value = numbers[5]; // PANIC!
    // thread 'main' panicked at 'index out of bounds:
    //   the len is 3 but the index is 5'
}
```

**Why it panics instead of returning garbage:** In C, accessing `numbers[5]` would read whatever random bytes happen to be in memory. This is a *buffer overread* — one of the most dangerous classes of security vulnerabilities. Rust eliminates this entire category of bugs by panicking instead.

**Safe alternative:**
```rust
fn main() {
    let numbers = vec![10, 20, 30];

    // .get() returns Option<&T> — no panic
    match numbers.get(5) {
        Some(val) => println!("Found: {}", val),
        None => println!("Index out of range"),
    }
}
```

### 2. Integer Overflow (Debug Mode)

```rust
fn main() {
    let x: u8 = 255;
    let y = x + 1; // PANIC in debug mode!
    // thread 'main' panicked at 'attempt to add with overflow'
}
```

In **release mode** (`cargo build --release`), integer overflow wraps around silently (255 + 1 = 0 for `u8`). In **debug mode**, it panics to catch bugs early.

**Safe alternatives:**
```rust
fn main() {
    let x: u8 = 255;

    // Explicit wrapping (like release mode)
    let a = x.wrapping_add(1);   // 0

    // Returns None on overflow
    let b = x.checked_add(1);    // None

    // Clamps at the maximum
    let c = x.saturating_add(1); // 255

    // Returns (result, overflowed)
    let (d, overflowed) = x.overflowing_add(1); // (0, true)

    println!("wrapping: {a}, checked: {b:?}, saturating: {c}, overflowing: ({d}, {overflowed})");
}
```

### 3. unwrap() on None or Err

```rust
fn main() {
    let name: Option<&str> = None;
    let value = name.unwrap(); // PANIC!
    // thread 'main' panicked at 'called `Option::unwrap()`
    //   on a `None` value'
}
```

```rust
fn main() {
    let result: Result<i32, &str> = Err("database offline");
    let value = result.unwrap(); // PANIC!
    // thread 'main' panicked at 'called `Result::unwrap()`
    //   on an `Err` value: "database offline"'
}
```

**Cousins of unwrap:**

```rust
fn main() {
    let x: Option<i32> = None;

    // unwrap() — generic panic message
    // x.unwrap();

    // expect() — YOUR custom panic message (preferred!)
    // x.expect("config must have a 'port' field");

    // unwrap_or() — provide a fallback, no panic
    let val = x.unwrap_or(8080);
    println!("Using port: {val}");
}
```

> **Rule of thumb:** Prefer `expect("why this should be Some/Ok")` over `unwrap()`.
> It makes the panic message actually useful when debugging.

### 4. Assertion Failures

```rust
fn main() {
    let result = 2 + 2;

    // assert! — checks a boolean condition
    assert!(result == 4);         // OK
    assert!(result == 5);         // PANIC: "assertion failed: result == 5"

    // assert_eq! — checks two values are equal
    assert_eq!(result, 4);        // OK
    assert_eq!(result, 5);        // PANIC: "assertion `left == right` failed
                                  //   left: 4, right: 5"

    // assert_ne! — checks two values are NOT equal
    assert_ne!(result, 5);        // OK
    assert_ne!(result, 4);        // PANIC: "assertion `left != right` failed
                                  //   left: 4, right: 4"
}
```

Assertions can include custom messages:

```rust
fn main() {
    let age = 15;
    assert!(
        age >= 18,
        "user must be at least 18, but age was {}",
        age
    );
    // PANIC: "user must be at least 18, but age was 15"
}
```

> **Note:** `assert!` runs in both debug and release builds. If you want assertions
> only in debug mode (for performance), use `debug_assert!`, `debug_assert_eq!`,
> and `debug_assert_ne!`.

### Quick Reference: Automatic Panics

```
┌──────────────────────────────┬──────────────────────────────────┐
│  Situation                   │  Example                         │
├──────────────────────────────┼──────────────────────────────────┤
│  Index out of bounds         │  vec[99] when len is 3           │
│  Integer overflow (debug)    │  255_u8 + 1                      │
│  unwrap() on None            │  None.unwrap()                   │
│  unwrap() on Err             │  Err("oops").unwrap()            │
│  assert! failure             │  assert!(false)                  │
│  assert_eq! failure          │  assert_eq!(1, 2)                │
│  Slice::from_raw_parts       │  invalid pointer / length        │
│  Division by zero (integers) │  42 / 0                          │
│  String::from invalid UTF-8  │  (via certain unsafe paths)      │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## Unwinding vs Aborting

When a panic happens, Rust has two strategies for shutting things down:

### Strategy 1: Unwinding (Default)

**Unwinding** means Rust walks back up the call stack, frame by frame, calling destructors (`Drop` implementations) for every value that's in scope. This cleans up resources properly.

```
    Call stack during unwinding
    ═══════════════════════════

    main()
      └─► process_data()
            └─► validate()
                  └─► check_bounds()
                        └─► PANIC! 💥

    Unwind direction: ◄──────────────────

    Step 1: Drop everything in check_bounds()
    Step 2: Drop everything in validate()
    Step 3: Drop everything in process_data()
    Step 4: Drop everything in main()
    Step 5: Process exits with error code
```

**What gets cleaned up during unwinding:**
- Files are closed (`File` implements `Drop`)
- Locks are released (`MutexGuard` implements `Drop`)
- Heap memory is freed (`Vec`, `String`, `Box` implement `Drop`)
- Network connections are closed
- Any custom `Drop` implementations run

```rust
struct DatabaseConn {
    name: String,
}

impl Drop for DatabaseConn {
    fn drop(&mut self) {
        println!("Closing database connection: {}", self.name);
    }
}

fn risky_operation() {
    let _conn = DatabaseConn {
        name: String::from("users_db"),
    };
    panic!("something broke!"); // _conn.drop() still runs!
}

fn main() {
    // We'll cover catch_unwind later — this catches the panic
    let result = std::panic::catch_unwind(|| {
        risky_operation();
    });
    println!("Panic caught: {:?}", result);
}
```

Output:
```
Closing database connection: users_db
Panic caught: Err(Any { .. })
```

Even though we panicked, the `DatabaseConn` destructor ran and cleaned up. That's unwinding at work.

### Strategy 2: Aborting

**Aborting** means the process immediately terminates. No destructors run, no stack unwinding, the OS simply reclaims all resources.

```
    Call stack during abort
    ═══════════════════════

    main()
      └─► process_data()
            └─► validate()
                  └─► check_bounds()
                        └─► PANIC! 💥

    ╔═══════════════════════════════════╗
    ║  PROCESS TERMINATED IMMEDIATELY   ║
    ║  No destructors. No cleanup.      ║
    ║  OS reclaims memory.              ║
    ╚═══════════════════════════════════╝
```

### How to Configure Abort on Panic

In your `Cargo.toml`:

```toml
# Abort only in release builds
[profile.release]
panic = 'abort'

# Abort in debug builds too (less common)
[profile.dev]
panic = 'abort'
```

### Comparison Table

| Aspect | Unwinding (default) | Aborting |
|---|---|---|
| **Destructors run?** | Yes | No |
| **Resources cleaned up?** | Yes, by Rust | By the OS (memory, file handles) |
| **Binary size** | Larger (unwinding tables) | Smaller (no unwinding machinery) |
| **Speed of exit** | Slower (walks the stack) | Instant |
| **catch_unwind works?** | Yes | No |
| **Good for** | Applications, servers, libraries | Embedded, WASM, small binaries |

### When to Choose Each

**Use unwinding (default) when:**
- You're writing a server that should log errors gracefully
- You need `catch_unwind` for thread isolation
- Your `Drop` implementations do important cleanup (flushing buffers, releasing locks)
- You're writing a library (let the consumer decide)

**Use abort when:**
- You're targeting embedded systems with limited memory
- You're compiling to WebAssembly
- Binary size matters more than graceful cleanup
- You never expect panics in production (and prefer a fast crash)

---

## Stack Traces and RUST_BACKTRACE

When a panic happens, Rust can show you the **full chain of function calls** that led to the panic. This is called a **backtrace** or **stack trace**.

### Enabling Backtraces

By default, Rust only shows the panic message and location. To see the full backtrace:

```
# Short backtrace (most useful — filters out noise)
$ RUST_BACKTRACE=1 cargo run

# Full backtrace (everything, including Rust internals)
$ RUST_BACKTRACE=full cargo run
```

On **Windows PowerShell**:
```powershell
$env:RUST_BACKTRACE=1; cargo run

# Or set it for the whole session:
$env:RUST_BACKTRACE = "1"
cargo run
```

### Reading a Backtrace

Here's what a backtrace looks like with `RUST_BACKTRACE=1`:

```
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99',
    src/main.rs:12:5
stack backtrace:
   0: rust_begin_unwind
             at /rustc/.../library/std/src/panicking.rs:665:5
   1: core::panicking::panic_fmt
             at /rustc/.../library/core/src/panicking.rs:76:14
   2: core::panicking::panic_bounds_check
             at /rustc/.../library/core/src/panicking.rs:234:5
   3: <usize as core::slice::index::SliceIndex<[T]>>::index
             at /rustc/.../library/core/src/slice/index.rs:255:10
   4: my_project::process_data        ◄── YOUR CODE
             at ./src/main.rs:12:5
   5: my_project::main                ◄── YOUR CODE
             at ./src/main.rs:6:5
   6: core::ops::function::FnOnce::call_once
             at /rustc/.../library/core/src/ops/function.rs:250:5
```

**How to read it:**

```
    Frame 0-3: Rust internals (ignore these)
                │
    Frame 4:    ◄── THIS IS WHERE THE PANIC HAPPENED
                │   my_project::process_data at src/main.rs:12
                │
    Frame 5:    ◄── THIS CALLED THE FUNCTION THAT PANICKED
                │   my_project::main at src/main.rs:6
                │
    Frame 6+:   Rust runtime (ignore these)
```

**Tips for reading backtraces:**
1. Look for frames with **your project name** or **your file paths**
2. The **lowest numbered frame** with your code is where the panic happened
3. The frames above it show the call chain that led there
4. Ignore frames from `std`, `core`, `rustc`, or `rust_begin_unwind`

### A Practical Example

```rust
fn get_username(id: usize) -> String {
    let users = vec!["Alice", "Bob", "Charlie"];
    users[id].to_string() // panics if id >= 3
}

fn greet_user(id: usize) {
    let name = get_username(id);
    println!("Hello, {}!", name);
}

fn main() {
    greet_user(99); // oops!
}
```

With `RUST_BACKTRACE=1`, the backtrace will point you to line 3 (`users[id]`) and show that `greet_user` called `get_username` which called the indexing operation. You can trace the bug straight to its source.

---

## Catching Panics with catch_unwind

Normally, a panic unwinds the stack and terminates the thread (or process). But sometimes you need to **catch** a panic and keep going. Rust provides `std::panic::catch_unwind` for this.

### Basic Usage

```rust
use std::panic;

fn main() {
    let result = panic::catch_unwind(|| {
        println!("About to panic...");
        panic!("oh no!");
    });

    match result {
        Ok(value) => println!("Closure returned: {:?}", value),
        Err(_) => println!("Caught a panic! Continuing..."),
    }

    println!("Program is still running!");
}
```

Output:
```
About to panic...
Caught a panic! Continuing...
Program is still running!
```

The panic was caught, and the program continued executing.

### What catch_unwind Returns

```rust
use std::panic;

fn main() {
    // Successful closure — returns Ok with the closure's return value
    let ok_result = panic::catch_unwind(|| {
        42
    });
    println!("{:?}", ok_result); // Ok(42)

    // Panicking closure — returns Err with an opaque Any value
    let err_result = panic::catch_unwind(|| -> i32 {
        panic!("kaboom");
    });
    println!("{:?}", err_result); // Err(Any { .. })
}
```

### Extracting the Panic Message

```rust
use std::panic;
use std::any::Any;

fn extract_panic_message(err: &Box<dyn Any + Send>) -> String {
    if let Some(s) = err.downcast_ref::<&str>() {
        s.to_string()
    } else if let Some(s) = err.downcast_ref::<String>() {
        s.clone()
    } else {
        "unknown panic payload".to_string()
    }
}

fn main() {
    let result = panic::catch_unwind(|| {
        panic!("something went wrong: code {}", 42);
    });

    if let Err(err) = result {
        let msg = extract_panic_message(&err);
        println!("Panic message: {}", msg);
    }
}
```

### When to Use catch_unwind

> **Important:** `catch_unwind` is **NOT** exception handling. Do not use it for
> normal error flow. It exists for specific, narrow use cases.

```
    ┌──────────────────────────────────────────────┐
    │         VALID USE CASES                       │
    ├──────────────────────────────────────────────┤
    │                                              │
    │  1. FFI Boundaries                           │
    │     Panics must NOT cross into C code.       │
    │     Catch them at the boundary.              │
    │                                              │
    │  2. Thread Pools / Task Runners              │
    │     One bad task shouldn't kill the pool.    │
    │     Catch panics to isolate failures.        │
    │                                              │
    │  3. Plugin Systems                           │
    │     Third-party code might panic.            │
    │     Protect the host application.            │
    │                                              │
    │  4. Test Harnesses                           │
    │     Verify that code panics when expected.   │
    │                                              │
    └──────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────┐
    │         INVALID USE CASES                    │
    ├──────────────────────────────────────────────┤
    │                                              │
    │  ✗ As a replacement for Result<T, E>         │
    │  ✗ For "expected" errors (file not found)    │
    │  ✗ For general control flow                  │
    │  ✗ As a try/catch like Java or Python        │
    │                                              │
    └──────────────────────────────────────────────┘
```

### FFI Boundary Example

```rust
use std::panic;

/// Called from C code — panics must NOT escape!
#[no_mangle]
pub extern "C" fn rust_function(input: i32) -> i32 {
    let result = panic::catch_unwind(|| {
        if input < 0 {
            panic!("negative input not supported");
        }
        input * 2
    });

    match result {
        Ok(value) => value,
        Err(_) => -1, // Return error code to C
    }
}
```

### Caveat: abort Mode Defeats catch_unwind

If you set `panic = 'abort'` in `Cargo.toml`, `catch_unwind` will **not** catch panics — the process aborts immediately. This is one of the key trade-offs between unwinding and aborting.

---

## The Panic Hook

A **panic hook** is a function that runs whenever a panic happens, *before* unwinding begins. You can use it to customize what happens when your program panics.

### Setting a Custom Hook

```rust
use std::panic;

fn main() {
    // Install a custom panic hook
    panic::set_hook(Box::new(|panic_info| {
        // panic_info contains details about the panic
        let location = panic_info.location().unwrap();
        let message = if let Some(s) = panic_info.payload().downcast_ref::<&str>() {
            s.to_string()
        } else if let Some(s) = panic_info.payload().downcast_ref::<String>() {
            s.clone()
        } else {
            "unknown".to_string()
        };

        eprintln!(
            "💥 PANIC at {}:{} — {}",
            location.file(),
            location.line(),
            message
        );
    }));

    panic!("test panic");
}
```

Output:
```
💥 PANIC at src/main.rs:21 — test panic
```

### Logging Panics in Production

In a real application, you might log panics to a file or crash reporting service:

```rust
use std::panic;
use std::fs::OpenOptions;
use std::io::Write;

fn setup_panic_logging() {
    panic::set_hook(Box::new(|info| {
        let location = info
            .location()
            .map(|l| format!("{}:{}", l.file(), l.line()))
            .unwrap_or_else(|| "unknown".into());

        let message = if let Some(s) = info.payload().downcast_ref::<&str>() {
            s.to_string()
        } else if let Some(s) = info.payload().downcast_ref::<String>() {
            s.clone()
        } else {
            "no message".to_string()
        };

        let log_line = format!(
            "[PANIC] location={} message=\"{}\"\n",
            location, message
        );

        // Write to crash log
        if let Ok(mut file) = OpenOptions::new()
            .create(true)
            .append(true)
            .open("crash.log")
        {
            let _ = file.write_all(log_line.as_bytes());
        }

        // Also print to stderr
        eprintln!("{}", log_line);
    }));
}

fn main() {
    setup_panic_logging();
    // ... rest of your application
    panic!("unexpected state in config parser");
}
```

### Getting Back the Default Hook

```rust
use std::panic;

fn main() {
    // Save and restore:
    let default_hook = panic::take_hook(); // removes current hook, returns it

    panic::set_hook(Box::new(|info| {
        eprintln!("Custom handler: {:?}", info);
    }));

    // Later, restore the default:
    panic::set_hook(default_hook);
}
```

---

## Panics Across Languages

Different languages take radically different approaches to unrecoverable errors. Understanding them helps you appreciate Rust's design.

### C: No Panic Mechanism

C has no built-in panic or exception mechanism. When something goes wrong:

```c
// C code — what happens on out-of-bounds access?
int arr[3] = {10, 20, 30};
int val = arr[99]; // Undefined Behavior!
// Could return garbage, could crash, could format your hard drive.
// The compiler is allowed to do LITERALLY ANYTHING.
```

C relies on the programmer to check everything manually. If they forget, the result is **undefined behavior** — silent corruption, security vulnerabilities, or segfaults.

### C++: Exceptions for Everything

C++ uses exceptions broadly — both for bugs and for expected errors:

```cpp
// C++ code
try {
    std::vector<int> v = {1, 2, 3};
    int val = v.at(99); // throws std::out_of_range
} catch (const std::out_of_range& e) {
    std::cout << "Caught: " << e.what() << std::endl;
}
```

C++ doesn't distinguish between "this is a bug" and "this is an expected condition." Both use the same `throw`/`catch` mechanism.

### Java: RuntimeException

Java comes closest to Rust's philosophy with its exception hierarchy:

```java
// Java code
// Checked exception — expected error (like Rust's Result)
throw new IOException("file not found");

// Unchecked exception — programming error (like Rust's panic)
throw new NullPointerException("this is a bug");
```

But in practice, Java developers often catch `RuntimeException` and continue, blurring the line.

### Go: panic/recover

Go has a `panic`/`recover` mechanism similar to Rust, but it's used more broadly:

```go
// Go code
func riskyOperation() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from:", r)
        }
    }()
    panic("something went wrong")
}
```

Go's `recover` is more commonly used than Rust's `catch_unwind`, partly because Go has no `Result` type — errors are returned as values alongside the result.

### Python: Exceptions for Everything

Python uses exceptions for all error types — there's no distinction between "bug" and "expected error":

```python
# Python code
try:
    result = int("not_a_number")  # Expected error
except ValueError:
    print("Invalid input")

try:
    x = my_list[99]  # Bug? Expected? Python doesn't distinguish
except IndexError:
    print("Index out of range")
```

### Comparison Table

```
┌────────────┬─────────────────┬────────────────────┬──────────────────┐
│  Language   │  "Bug" errors   │  "Expected" errors │  Distinction?    │
├────────────┼─────────────────┼────────────────────┼──────────────────┤
│  Rust       │  panic!         │  Result<T, E>      │  ✓ Very clear    │
│  C          │  Undefined      │  Error codes       │  ✗ No mechanism  │
│             │  behavior       │  (return -1, etc.) │    for bugs      │
│  C++        │  throw          │  throw             │  ✗ Same mechanism│
│  Java       │  RuntimeExc.    │  Checked Exc.      │  ~ Partially     │
│  Go         │  panic()        │  return err        │  ~ Somewhat      │
│  Python     │  raise          │  raise             │  ✗ Same mechanism│
│  Swift      │  fatalError()   │  throws / Result   │  ✓ Clear         │
│  Haskell    │  error/undefined│  Maybe/Either      │  ✓ Clear         │
└────────────┴─────────────────┴────────────────────┴──────────────────┘
```

**Rust's position:** Rust is one of the few languages that makes a **hard, compiler-visible distinction** between "this is a bug" (panic) and "this might fail" (Result). This makes error handling more intentional and less error-prone.

---

## Panic in Multi-threaded Code

One of Rust's most important guarantees: **a panic in one thread does not crash the entire process** (unless it's the main thread).

### Thread Isolation

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        panic!("worker thread crashed!");
    });

    // The main thread is still alive!
    println!("Main thread is fine.");

    // JoinHandle::join() returns Err if the thread panicked
    match handle.join() {
        Ok(_) => println!("Thread finished successfully"),
        Err(e) => println!("Thread panicked: {:?}", e),
    }

    println!("Program continues after thread panic.");
}
```

Output:
```
Main thread is fine.
thread '<unnamed>' panicked at 'worker thread crashed!', src/main.rs:5:9
Thread panicked: Any { .. }
Program continues after thread panic.
```

### How It Works

```
    ┌──────────────────────────────────────────┐
    │              Main Thread                  │
    │                                          │
    │  spawn ──┬──► Worker Thread              │
    │          │                               │
    │          │    ┌──────────────────────┐    │
    │          │    │ panic!("crashed!")   │    │
    │          │    │                      │    │
    │          │    │ Unwind THIS thread   │    │
    │          │    │ only. Drop all       │    │
    │          │    │ values in this       │    │
    │          │    │ thread's stack.      │    │
    │          │    └──────────┬───────────┘    │
    │          │               │                │
    │  join() ◄────────────────┘                │
    │  Returns Err(...)                         │
    │                                          │
    │  Main thread continues normally!          │
    └──────────────────────────────────────────┘
```

### But Main Thread Panics Kill Everything

```rust
fn main() {
    let handle = thread::spawn(|| {
        thread::sleep(std::time::Duration::from_secs(5));
        println!("Worker done"); // Never prints!
    });

    panic!("main thread panicked!"); // Entire process exits

    // This line is never reached
    handle.join().unwrap();
}
```

When the main thread panics, the process exits, and all other threads are terminated.

### Practical Pattern: Resilient Worker Pool

```rust
use std::thread;

fn process_job(job_id: usize) {
    if job_id == 3 {
        panic!("job {} is cursed!", job_id);
    }
    println!("Job {} completed successfully", job_id);
}

fn main() {
    let mut handles = vec![];

    for id in 1..=5 {
        let handle = thread::spawn(move || {
            process_job(id);
        });
        handles.push((id, handle));
    }

    let mut failures = vec![];
    for (id, handle) in handles {
        match handle.join() {
            Ok(_) => println!("✓ Job {} succeeded", id),
            Err(_) => {
                println!("✗ Job {} panicked", id);
                failures.push(id);
            }
        }
    }

    if failures.is_empty() {
        println!("\nAll jobs completed successfully!");
    } else {
        println!("\nFailed jobs: {:?}", failures);
    }
}
```

Output (order may vary):
```
Job 1 completed successfully
Job 2 completed successfully
thread '<unnamed>' panicked at 'job 3 is cursed!', src/main.rs:4:9
Job 4 completed successfully
Job 5 completed successfully
✓ Job 1 succeeded
✓ Job 2 succeeded
✗ Job 3 panicked
✓ Job 4 succeeded
✓ Job 5 succeeded

Failed jobs: [3]
```

Job 3 panicked, but every other job completed fine. The main thread collected results and reported the failure.

---

## When to Use panic! (Guidelines)

### ✓ When panic IS Appropriate

#### 1. Prototyping and Quick Scripts

When you're exploring an idea, `unwrap()` is perfectly fine:

```rust
fn main() {
    // Quick prototype — not production code
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
    println!("Starting on port {port}");
}
```

This is fine for prototypes. You'll replace the `unwrap()`s with proper error handling before shipping.

#### 2. Violated Invariants (Bugs)

If a condition represents a **bug** that should never happen with correct code:

```rust
fn process_direction(code: u8) {
    // This function's contract: code is 0-3
    // If it's not, that's a bug in the CALLER
    match code {
        0 => println!("North"),
        1 => println!("South"),
        2 => println!("East"),
        3 => println!("West"),
        _ => panic!(
            "BUG: invalid direction code {}. \
             This indicates a logic error in the caller.",
            code
        ),
    }
}
```

#### 3. When Continuing Would Cause Worse Problems

```rust
fn initialize_critical_system() -> CriticalSystem {
    let key = std::env::var("ENCRYPTION_KEY")
        .expect("ENCRYPTION_KEY must be set — cannot start without it");

    // If encryption key is missing, better to crash NOW
    // than to silently process unencrypted data
    CriticalSystem::new(&key)
}
```

#### 4. Tests and Examples

In tests, `unwrap()` and `assert!` are the right tools:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_parse_valid_input() {
        let result = parse("hello world").unwrap(); // unwrap is fine in tests
        assert_eq!(result.word_count, 2);
        assert_eq!(result.char_count, 11);
    }

    #[test]
    #[should_panic(expected = "empty input")]
    fn test_parse_empty_panics() {
        parse(""); // Should panic
    }
}
```

### ✗ When panic is NOT Appropriate

#### 1. Never in Library Code for Expected Errors

```rust
// BAD — library panics on bad input
pub fn parse_age(input: &str) -> u32 {
    input.parse().unwrap() // DON'T DO THIS
}

// GOOD — library returns Result
pub fn parse_age(input: &str) -> Result<u32, ParseAgeError> {
    let age: u32 = input.parse().map_err(|_| ParseAgeError::NotANumber)?;
    if age > 150 {
        return Err(ParseAgeError::Unrealistic(age));
    }
    Ok(age)
}
```

#### 2. Never for User Input Errors

```rust
// BAD — panic on bad user input
fn get_selection(input: &str) -> MenuItem {
    let n: usize = input.trim().parse().unwrap(); // DON'T
    MENU_ITEMS[n] // DON'T — could be out of bounds
}

// GOOD — return Result, let caller decide
fn get_selection(input: &str) -> Result<MenuItem, SelectionError> {
    let n: usize = input.trim().parse()
        .map_err(|_| SelectionError::NotANumber)?;
    MENU_ITEMS.get(n)
        .copied()
        .ok_or(SelectionError::OutOfRange(n))
}
```

#### 3. Never for I/O or Network Errors

```rust
// BAD — panic on network error
fn fetch_data(url: &str) -> String {
    reqwest::blocking::get(url).unwrap().text().unwrap() // DON'T
}

// GOOD — propagate the error
fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let response = reqwest::blocking::get(url)?;
    let text = response.text()?;
    Ok(text)
}
```

### Decision Flowchart

```
    Is this situation a BUG in the code?
    │
    ├── YES ──► Is it a violated invariant or logic error?
    │           │
    │           ├── YES ──► panic! ✓
    │           │
    │           └── NO ───► Could the caller have prevented it?
    │                       │
    │                       ├── YES ──► Document the contract, panic! ✓
    │                       └── NO ───► Use Result
    │
    └── NO ───► Is it an expected failure? (I/O, parsing, network)
                │
                ├── YES ──► Use Result ✓
                │
                └── NO ───► Are you prototyping?
                            │
                            ├── YES ──► unwrap() is fine for now ✓
                            └── NO ───► Use Result ✓
```

---

## Exercises

### Exercise 1: Panic Detective

The following program has several places that could panic. Identify each one and fix it to use proper error handling instead.

```rust
use std::collections::HashMap;

fn get_config_value(config: &HashMap<String, String>, key: &str) -> String {
    config[key].clone() // Hint: what if key doesn't exist?
}

fn parse_port(s: &str) -> u16 {
    s.parse().unwrap() // Hint: what if s is "abc"?
}

fn get_first_word(text: &str) -> &str {
    let words: Vec<&str> = text.split_whitespace().collect();
    words[0] // Hint: what if text is empty?
}

fn main() {
    let mut config = HashMap::new();
    config.insert("host".to_string(), "localhost".to_string());

    println!("Host: {}", get_config_value(&config, "host"));
    println!("Port: {}", parse_port("8080"));
    println!("First word: {}", get_first_word("hello world"));

    // These would panic! Fix the functions so they don't.
    // println!("Missing: {}", get_config_value(&config, "missing_key"));
    // println!("Bad port: {}", parse_port("not_a_number"));
    // println!("Empty: {}", get_first_word(""));
}
```

**Your task:**
1. Change each function to return `Result<T, E>` or `Option<T>` instead of panicking
2. Handle the error cases in `main()` gracefully
3. Uncomment the three panic-triggering lines and make them work without panicking

### Exercise 2: Custom Panic Hook

Write a program that:
1. Sets a custom panic hook that writes panic info to a `String` (use `Arc<Mutex<String>>` to share it)
2. Spawns 3 threads — one of which will panic
3. After all threads complete, prints the collected panic information

Starter code:

```rust
use std::panic;
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let panic_log = Arc::new(Mutex::new(String::new()));

    // TODO: Set a custom panic hook that appends to panic_log

    let mut handles = vec![];

    for i in 0..3 {
        let handle = thread::spawn(move || {
            if i == 1 {
                panic!("Thread {} encountered a critical error", i);
            }
            println!("Thread {} completed", i);
        });
        handles.push(handle);
    }

    for handle in handles {
        let _ = handle.join();
    }

    // TODO: Print the contents of panic_log
}
```

### Exercise 3: Unwinding Verifier

Write a program that proves destructors run during unwinding:

1. Create a struct `Resource` with a `name: String` field
2. Implement `Drop` for `Resource` that prints `"Dropping: {name}"`
3. Write a function that creates three `Resource` values, then panics after the second
4. Use `catch_unwind` to catch the panic
5. Verify (by reading the output) that all three resources created before the panic were dropped

Expected output:
```
Creating: Alpha
Creating: Beta
Creating: Gamma
About to panic!
Dropping: Gamma
Dropping: Beta
Dropping: Alpha
Caught panic! All resources were cleaned up.
```

---

## Summary

| Concept | Key Point |
|---|---|
| **What is a panic** | Rust's mechanism for unrecoverable errors (bugs) |
| **panic! macro** | Triggers a panic with a formatted message |
| **Automatic panics** | Out-of-bounds, overflow, unwrap on None/Err, assertions |
| **Unwinding** | Default — walks stack, runs destructors, cleans up |
| **Aborting** | Alternative — instant exit, no cleanup, smaller binary |
| **RUST_BACKTRACE** | Set to `1` or `full` to see the call stack on panic |
| **catch_unwind** | Catches panics — use for FFI, thread pools, not error handling |
| **Panic hook** | Custom function that runs before unwinding starts |
| **Threads** | Panic in one thread doesn't crash others — checked via join() |
| **When to panic** | Bugs, invariant violations, prototypes — NOT expected errors |
| **vs other languages** | Rust uniquely separates bugs (panic) from errors (Result) |

> **The golden rule:** If it's the *caller's* fault (bad input, missing file), return
> a `Result`. If it's the *programmer's* fault (impossible state, violated invariant),
> `panic!`.

---

**Previous:** [← Error Philosophy](./01-error-philosophy.md) · **Next:** [Result in Depth →](./03-result-in-depth.md)

<p align="center"><i>Tutorial 2 of 7 — Stage 7: Error Handling</i></p>

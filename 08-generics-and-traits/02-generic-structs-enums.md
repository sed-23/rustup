# Generic Structs & Enums 🎯

> **"Generics let you write one data structure that works with any type — the compiler generates specialized versions for each type you actually use, with zero runtime cost."**

---

## Table of Contents

- [Generic Structs](#generic-structs)
- [Multiple Type Parameters](#multiple-type-parameters)
- [Methods on Generic Structs](#methods-on-generic-structs)
- [Specialized Methods](#specialized-methods)
- [Generic Enums](#generic-enums)
- [Generic Enums You Can Build](#generic-enums-you-can-build)
- [Memory Layout of Generic Types](#memory-layout-of-generic-types)
- [Generics in Standard Library Data Structures](#generics-in-standard-library-data-structures)
- [The Turbofish Syntax](#the-turbofish-syntax)
- [PhantomData — Generics Without Data](#phantomdata--generics-without-data)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Generic Structs

In the previous tutorial we parameterized *functions* over types. The same idea
applies to **structs** — define one blueprint and let the compiler stamp out
concrete versions for every type you plug in.

### The Problem Without Generics

Suppose you want a 2-D point. Without generics you'd end up with duplicated
definitions:

```rust
struct PointI32 {
    x: i32,
    y: i32,
}

struct PointF64 {
    x: f64,
    y: f64,
}

// PointU8, PointI16, PointF32 ... the list never ends
```

That's tedious, error-prone, and impossible to maintain.

### One Struct to Rule Them All

A **generic struct** introduces a type parameter between angle brackets:

```rust
struct Point<T> {
    x: T,
    y: T,
}
```

`T` is a placeholder. Wherever `T` appears, the *same* concrete type will be
substituted when you create a value:

```rust
fn main() {
    let integer_point = Point { x: 5, y: 10 };       // Point<i32>
    let float_point   = Point { x: 1.0, y: 4.5 };    // Point<f64>

    println!("int:   ({}, {})", integer_point.x, integer_point.y);
    println!("float: ({}, {})", float_point.x, float_point.y);
}
```

Rust **infers** `T` from the values you pass. You don't have to write
`Point::<i32> { ... }` (though you can — see [Turbofish](#the-turbofish-syntax)
later).

### Both Fields Must Match

Because **both** `x` and `y` are typed `T`, they must be the same type:

```rust
// ERROR — mismatched types
let p = Point { x: 5, y: 4.0 };
//                        ^^^ expected integer, found float
```

If you need mixed types, read on.

---

## Multiple Type Parameters

Add a second type parameter to allow each field to carry a different type:

```rust
struct Point<T, U> {
    x: T,
    y: U,
}
```

Now every combination is legal:

```rust
fn main() {
    let both_int   = Point { x: 5, y: 10 };        // Point<i32, i32>
    let both_float = Point { x: 1.0, y: 4.0 };     // Point<f64, f64>
    let mixed      = Point { x: 5, y: 4.0 };        // Point<i32, f64>
    let also_mixed = Point { x: 'a', y: true };      // Point<char, bool>

    println!("mixed: ({}, {})", mixed.x, mixed.y);
}
```

### Convention for Type Parameter Names

| Letter | Typical meaning            |
|--------|----------------------------|
| `T`    | first / primary **T**ype   |
| `U`    | second type                |
| `V`    | third type                 |
| `K`    | **K**ey (in maps)          |
| `V`    | **V**alue (in maps)        |
| `E`    | **E**rror                  |

You're free to use any identifier (`MyType`, `Item`, etc.), but single
upper-case letters are idiomatic for generic parameters.

### How Many Parameters Are Too Many?

There is no hard limit, but **more than 2–3** usually signals a design problem.
If you find yourself writing `struct Foo<A, B, C, D, E>`, consider grouping
related parameters into their own struct.

---

## Methods on Generic Structs

To define methods on a generic struct, you write `impl<T>` — the `<T>` after
`impl` *declares* the type parameter so it can be used in the block:

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    // Available for ALL types T
    fn x(&self) -> &T {
        &self.x
    }

    fn y(&self) -> &T {
        &self.y
    }

    fn new(x: T, y: T) -> Self {
        Point { x, y }
    }
}

fn main() {
    let p = Point::new(3, 7);
    println!("p.x = {}", p.x());
    println!("p.y = {}", p.y());
}
```

> **Why `impl<T>` and not just `impl`?**
>
> The `<T>` after `impl` tells Rust: "this block is generic over some type `T`."
> Without it, Rust would look for a **concrete** type literally named `T`, which
> doesn't exist.

### Methods on Structs With Multiple Parameters

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn x(&self) -> &T {
        &self.x
    }

    fn y(&self) -> &U {
        &self.y
    }

    /// Creates a NEW point by mixing the x of `self` with the y of `other`.
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };       // Point<i32, f64>
    let p2 = Point { x: "Hello", y: 'c' };   // Point<&str, char>

    let p3 = p1.mixup(p2);                    // Point<i32, char>
    println!("p3 = ({}, {})", p3.x, p3.y);    // p3 = (5, c)
}
```

Notice that `mixup` introduces its **own** additional type parameters (`V`, `W`)
that are independent of the struct's `T` and `U`. Methods can be generic too!

---

## Specialized Methods

You can write an `impl` block for a **specific** concrete type instead of the
generic `T`. These methods will only be available when the struct is
instantiated with that type.

```rust
struct Point<T> {
    x: T,
    y: T,
}

// Generic — available for every Point<T>
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point { x, y }
    }
}

// Specialized — ONLY available for Point<f64>
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

fn main() {
    let float_point = Point::new(3.0, 4.0);
    println!("distance = {}", float_point.distance_from_origin()); // 5.0

    let int_point = Point::new(3, 4);
    // int_point.distance_from_origin();  // ERROR — method not found for Point<i32>
}
```

This is incredibly powerful — you get shared behaviour in the generic block
**and** type-specific behaviour where it makes sense, without any runtime
dispatch.

### Why Not Just Use a Trait Bound?

You *could* require `T: Float` (a hypothetical trait) to enable math methods,
but specialized `impl` blocks let you add methods for a **single** type with
no trait in sight. Both approaches have their place:

| Technique                   | When to use                                          |
|-----------------------------|------------------------------------------------------|
| `impl<T: SomeTrait> Foo<T>` | When many types share a capability via a trait       |
| `impl Foo<ConcreteType>`    | When you want behaviour for exactly one type         |

---

## Generic Enums

You've been using generic enums since your very first Rust program — you just
might not have realized it.

### Option&lt;T&gt; — The Most Famous Generic Enum

```rust
// Simplified from std — this is what the standard library defines
enum Option<T> {
    Some(T),
    None,
}
```

`Option<T>` is generic over `T`. When you write `Some(42)`, the compiler infers
`Option<i32>`. When you write `Some("hello")`, it infers `Option<&str>`.

```rust
fn find_first_even(numbers: &[i32]) -> Option<i32> {
    for &n in numbers {
        if n % 2 == 0 {
            return Some(n);
        }
    }
    None
}

fn main() {
    let nums = vec![1, 3, 5, 8, 13];
    match find_first_even(&nums) {
        Some(n) => println!("First even: {n}"),
        None    => println!("No even numbers found"),
    }
}
```

Every time you use `Some` or `None` you're using generics. **You've been using
generics all along!**

### Result&lt;T, E&gt; — Two Type Parameters

```rust
// Simplified from std
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Result` carries **two** type parameters — one for the success value and one
for the error:

```rust
use std::num::ParseIntError;

fn parse_number(s: &str) -> Result<i32, ParseIntError> {
    s.trim().parse::<i32>()
}

fn main() {
    match parse_number("42") {
        Ok(n)  => println!("Parsed: {n}"),
        Err(e) => println!("Error: {e}"),
    }

    match parse_number("abc") {
        Ok(n)  => println!("Parsed: {n}"),
        Err(e) => println!("Error: {e}"),
    }
}
```

### How to Read Generic Enum Definitions

When you see a generic enum, read it as:

```
enum EnumName<TypeParam1, TypeParam2, ...> {
    Variant1(uses TypeParam1),
    Variant2(uses TypeParam2),
    Variant3,                    // not every variant must use a parameter
}
```

Each variant can use **any** of the declared type parameters — or none at all
(like `None` in `Option<T>`).

---

## Generic Enums You Can Build

The standard library's `Option` and `Result` aren't magic. You can define your
own generic enums just as easily.

### Either&lt;L, R&gt; — A Choice Between Two Types

```rust
enum Either<L, R> {
    Left(L),
    Right(R),
}

impl<L, R> Either<L, R> {
    fn is_left(&self) -> bool {
        matches!(self, Either::Left(_))
    }

    fn is_right(&self) -> bool {
        matches!(self, Either::Right(_))
    }
}

fn divide(a: f64, b: f64) -> Either<f64, String> {
    if b == 0.0 {
        Either::Right(String::from("division by zero"))
    } else {
        Either::Left(a / b)
    }
}

fn main() {
    let result = divide(10.0, 3.0);
    match result {
        Either::Left(val)  => println!("Result: {val}"),
        Either::Right(msg) => println!("Error: {msg}"),
    }
}
```

> `Either` is essentially a simpler, unbiased version of `Result`. In practice,
> prefer `Result` for operations that can fail, but `Either` is useful when
> neither side represents an "error."

### Tree&lt;T&gt; — A Recursive Generic Type

```rust
enum Tree<T> {
    Leaf(T),
    Node(Box<Tree<T>>, Box<Tree<T>>),
}

impl<T: std::fmt::Display> Tree<T> {
    fn sum_display(&self, depth: usize) {
        let indent = "  ".repeat(depth);
        match self {
            Tree::Leaf(val) => println!("{indent}Leaf({val})"),
            Tree::Node(left, right) => {
                println!("{indent}Node");
                left.sum_display(depth + 1);
                right.sum_display(depth + 1);
            }
        }
    }
}

fn main() {
    //        Node
    //       /    \
    //     Node   Leaf(3)
    //    /    \
    // Leaf(1) Leaf(2)

    let tree = Tree::Node(
        Box::new(Tree::Node(
            Box::new(Tree::Leaf(1)),
            Box::new(Tree::Leaf(2)),
        )),
        Box::new(Tree::Leaf(3)),
    );

    tree.sum_display(0);
}
```

### Why Box Is Needed for Recursive Types

Without `Box`, the compiler would try to compute the size of `Tree<T>` and
find that it contains another `Tree<T>`, which contains another `Tree<T>`,
ad infinitum — **infinite size**.

```
// Without Box — this does NOT compile:
enum Tree<T> {
    Leaf(T),
    Node(Tree<T>, Tree<T>),   // ERROR: recursive type has infinite size
}
```

`Box<T>` is a **heap-allocated pointer** with a fixed size (8 bytes on 64-bit
systems), which breaks the infinite recursion:

```
Tree<T> size with Box:
┌──────────────────────────────────────────┐
│ Leaf variant:  [tag: 8 bytes][T: N bytes]│
│ Node variant:  [tag: 8 bytes][Box: 8][Box: 8] │
└──────────────────────────────────────────┘
The compiler can compute a finite size — done!
```

---

## Memory Layout of Generic Types

One of Rust's most impressive properties is **monomorphization** — the compiler
generates a separate, fully optimized copy of each generic type for every
concrete type you use. There is **zero** runtime overhead.

### Concrete Sizes

```rust
use std::mem::size_of;

struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    println!("Point<i32>: {} bytes", size_of::<Point<i32>>());   // 8
    println!("Point<f64>: {} bytes", size_of::<Point<f64>>());   // 16
    println!("Point<u8>:  {} bytes", size_of::<Point<u8>>());    // 2
    println!("Point<i64>: {} bytes", size_of::<Point<i64>>());   // 16
}
```

Each concrete instantiation has its own memory layout:

```
Point<i32> — 8 bytes total
┌─────────┬─────────┐
│  x: i32 │  y: i32 │
│ 4 bytes  │ 4 bytes  │
└─────────┴─────────┘

Point<f64> — 16 bytes total
┌──────────┬──────────┐
│  x: f64  │  y: f64  │
│ 8 bytes   │ 8 bytes   │
└──────────┴──────────┘

Point<u8> — 2 bytes total
┌────────┬────────┐
│ x: u8  │ y: u8  │
│ 1 byte  │ 1 byte  │
└────────┴────────┘
```

### Monomorphization Under the Hood

When you write:

```rust
let a = Point { x: 5_i32, y: 10 };
let b = Point { x: 1.0_f64, y: 2.0 };
```

The compiler essentially generates:

```rust
// Auto-generated by the compiler (you never see this)
struct Point_i32 { x: i32, y: i32 }
struct Point_f64 { x: f64, y: f64 }

let a = Point_i32 { x: 5, y: 10 };
let b = Point_f64 { x: 1.0, y: 2.0 };
```

Each version is compiled and optimized independently — identical performance to
hand-written structs. **Generics are a compile-time abstraction with zero
runtime cost.**

### Option&lt;T&gt; and Niche Optimization

Rust performs a remarkable optimization for `Option` wrapping certain types.
Consider `Option<&T>`:

```rust
use std::mem::size_of;

fn main() {
    // A reference is 8 bytes (on 64-bit)
    println!("&i32:          {} bytes", size_of::<&i32>());          // 8
    // Option<&i32> is ALSO 8 bytes!
    println!("Option<&i32>:  {} bytes", size_of::<Option<&i32>>());  // 8

    // Compare with Option<i32>: needs an extra byte for the tag
    println!("i32:           {} bytes", size_of::<i32>());           // 4
    println!("Option<i32>:   {} bytes", size_of::<Option<i32>>());   // 8
}
```

How is this possible? References can **never** be null in Rust. So the compiler
uses the null pointer value (`0x0`) to represent `None`, and any non-zero
pointer to represent `Some(&T)`. This is called **niche optimization**:

```
Option<&T> — 8 bytes (no overhead!)
┌────────────────────────────┐
│ 0x0000_0000_0000_0000      │  ← None
│ 0x7fff_abcd_1234_5678      │  ← Some(&T)
└────────────────────────────┘
The null pointer IS the None variant — no tag needed.

Option<i32> — 8 bytes (4 tag + 4 data, with padding)
┌──────────────┬──────────────┐
│ discriminant  │   value      │
│  (4 bytes)    │  i32 (4 bytes)│
└──────────────┴──────────────┘
```

This means wrapping a reference in `Option` is **completely free** in terms of
memory. The same optimization applies to `Box<T>`, `NonZeroU32`, and other
types that have "niches" — invalid bit patterns the compiler can repurpose.

---

## Generics in Standard Library Data Structures

The standard library is **deeply generic**. Nearly every collection and
container is parameterized:

| Type                | Parameters | Description                        |
|---------------------|------------|------------------------------------|
| `Vec<T>`            | `T`        | Growable array                     |
| `HashMap<K, V>`     | `K`, `V`   | Hash-based key-value map           |
| `BTreeMap<K, V>`    | `K`, `V`   | Sorted key-value map (B-tree)      |
| `HashSet<T>`        | `T`        | Hash-based unique set              |
| `BTreeSet<T>`       | `T`        | Sorted unique set                  |
| `LinkedList<T>`     | `T`        | Doubly-linked list                 |
| `VecDeque<T>`       | `T`        | Double-ended queue                 |
| `Box<T>`            | `T`        | Heap-allocated pointer             |
| `Rc<T>`             | `T`        | Reference-counted pointer          |
| `Arc<T>`            | `T`        | Atomic reference-counted pointer   |
| `Cell<T>`           | `T`        | Interior mutability (Copy types)   |
| `RefCell<T>`        | `T`        | Interior mutability (runtime check)|

### Simplified Vec&lt;T&gt; Definition

Here's a stripped-down version of how `Vec<T>` is defined in the standard
library:

```rust
// Simplified — real Vec has more fields and unsafe code
pub struct Vec<T> {
    ptr: *mut T,     // raw pointer to heap-allocated buffer
    len: usize,      // number of elements currently stored
    cap: usize,      // total capacity of the buffer
}

impl<T> Vec<T> {
    pub fn new() -> Self {
        Vec {
            ptr: std::ptr::null_mut(),
            len: 0,
            cap: 0,
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    // push, pop, indexing, etc. all defined generically
}
```

Every method works for *any* `T` — `Vec<i32>`, `Vec<String>`,
`Vec<Vec<HashMap<String, bool>>>` — all from one definition.

### Nesting Generics

Generics compose freely:

```rust
use std::collections::HashMap;

fn main() {
    // A HashMap whose keys are Strings and values are Vec<i32>
    let mut scores: HashMap<String, Vec<i32>> = HashMap::new();

    scores.entry("Alice".to_string()).or_default().push(95);
    scores.entry("Alice".to_string()).or_default().push(87);
    scores.entry("Bob".to_string()).or_default().push(72);

    for (name, vals) in &scores {
        println!("{name}: {vals:?}");
    }
}
```

---

## The Turbofish Syntax

Sometimes Rust **can't infer** the type parameter from context. The
**turbofish** syntax — `::<Type>` — lets you specify it explicitly.

### The Name

The turbofish is named after its visual resemblance to a fish:

```
::<>    ← looks a bit like a fish swimming to the right, right?
```

### When You Need It

**Parsing strings** is the most common case:

```rust
fn main() {
    // Rust can't tell what type to parse into — turbofish to the rescue!
    let num = "42".parse::<i32>().unwrap();
    println!("{num}");

    // Alternative: annotate the binding instead
    let num: i32 = "42".parse().unwrap();

    // Both are equivalent — choose whichever reads better
}
```

**Creating empty collections** is another common situation:

```rust
fn main() {
    // Without type annotation, Rust doesn't know what T is
    let v = Vec::<i32>::new();

    // Equivalent alternatives:
    let v: Vec<i32> = Vec::new();
    let v = vec![0_i32];   // inferred from the element
}
```

### Turbofish on Functions

You can turbofish any generic function:

```rust
fn make_pair<T: Default>() -> (T, T) {
    (T::default(), T::default())
}

fn main() {
    let pair = make_pair::<String>();
    println!("{:?}", pair);  // ("", "")

    let pair = make_pair::<i32>();
    println!("{:?}", pair);  // (0, 0)

    let pair = make_pair::<bool>();
    println!("{:?}", pair);  // (false, false)
}
```

### Turbofish on Methods

When a method itself has generic parameters:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // collect() is generic — turbofish specifies the output collection type
    let doubled: Vec<i32> = numbers.iter().map(|&x| x * 2).collect();

    // Or with turbofish:
    let doubled = numbers.iter().map(|&x| x * 2).collect::<Vec<i32>>();

    // Partial turbofish — let Rust infer the inner type:
    let doubled = numbers.iter().map(|&x| x * 2).collect::<Vec<_>>();

    println!("{doubled:?}");
}
```

> **Rule of thumb:** prefer type annotations on `let` bindings for readability.
> Use turbofish when there's no binding to annotate (e.g., in the middle of a
> method chain).

---

## PhantomData — Generics Without Data

Sometimes you want a struct to be generic over a type parameter **that doesn't
appear in any field**. This sounds strange, but it comes up in advanced
patterns.

### The Problem

```rust
struct Meters<T> {
    value: f64,
    // T is declared but never used — ERROR!
}
```

```
error[E0392]: parameter `T` is never used
 --> src/main.rs:1:15
  |
1 | struct Meters<T> {
  |               ^ unused parameter
```

Rust won't allow unused type parameters because it needs to know how the type
relates to `T` for memory layout and drop-checking purposes.

### The Solution: PhantomData

`std::marker::PhantomData<T>` is a **zero-sized type** that tells the compiler
"pretend this struct uses `T`":

```rust
use std::marker::PhantomData;

struct Meters<T> {
    value: f64,
    _unit: PhantomData<T>,   // zero-sized, no runtime cost
}

struct Metric;
struct Imperial;

impl<T> Meters<T> {
    fn new(value: f64) -> Self {
        Meters {
            value,
            _unit: PhantomData,
        }
    }
}

fn main() {
    let m = Meters::<Metric>::new(100.0);
    let i = Meters::<Imperial>::new(328.084);

    // m and i are DIFFERENT types — you can't accidentally mix them
    // even though they have the same runtime representation
    println!("Metric:   {}", m.value);
    println!("Imperial: {}", i.value);
}
```

### Why PhantomData Matters

- **Type-state patterns** — encode state machine transitions at the type level
- **Lifetime markers** — `PhantomData<&'a T>` tells the compiler a struct
  borrows from `'a` even if no field stores a reference
- **Unit markers** — prevent mixing meters with feet, seconds with milliseconds

`PhantomData` has zero size and zero runtime cost:

```rust
use std::mem::size_of;
use std::marker::PhantomData;

struct Tagged<T> {
    value: i32,
    _marker: PhantomData<T>,
}

fn main() {
    println!("i32:       {} bytes", size_of::<i32>());             // 4
    println!("Tagged<()>: {} bytes", size_of::<Tagged<()>>());      // 4 — same!
    println!("PhantomData<i32>: {} bytes", size_of::<PhantomData<i32>>()); // 0
}
```

> **You don't need to master PhantomData right now.** It's an advanced tool, but
> knowing it exists helps you understand code in the standard library and
> third-party crates.

---

## Exercises

### Exercise 1: Generic Pair With a Swap Method

Define a struct `Pair<T>` that holds two values of the same type. Implement:

- `fn new(first: T, second: T) -> Self`
- `fn first(&self) -> &T`
- `fn second(&self) -> &T`
- `fn swap(self) -> Pair<T>` — returns a new Pair with the values swapped

```rust
// TODO: Define Pair<T> and implement the methods described above

fn main() {
    let p = Pair::new(1, 2);
    println!("({}, {})", p.first(), p.second()); // (1, 2)

    let swapped = p.swap();
    println!("({}, {})", swapped.first(), swapped.second()); // (2, 1)

    let p2 = Pair::new("hello", "world");
    let swapped2 = p2.swap();
    println!("({}, {})", swapped2.first(), swapped2.second()); // (world, hello)
}
```

<details>
<summary>✅ Solution</summary>

```rust
struct Pair<T> {
    first: T,
    second: T,
}

impl<T> Pair<T> {
    fn new(first: T, second: T) -> Self {
        Pair { first, second }
    }

    fn first(&self) -> &T {
        &self.first
    }

    fn second(&self) -> &T {
        &self.second
    }

    fn swap(self) -> Pair<T> {
        Pair {
            first: self.second,
            second: self.first,
        }
    }
}

fn main() {
    let p = Pair::new(1, 2);
    println!("({}, {})", p.first(), p.second()); // (1, 2)

    let swapped = p.swap();
    println!("({}, {})", swapped.first(), swapped.second()); // (2, 1)

    let p2 = Pair::new("hello", "world");
    let swapped2 = p2.swap();
    println!("({}, {})", swapped2.first(), swapped2.second()); // (world, hello)
}
```

</details>

---

### Exercise 2: Generic Stack Enum

Build a simple stack using a linked-list-style enum:

```rust
enum Stack<T> {
    Empty,
    Entry(T, Box<Stack<T>>),
}
```

Implement:

- `fn new() -> Self` — creates an empty stack
- `fn push(self, item: T) -> Self` — pushes an item, returns a new stack
- `fn peek(&self) -> Option<&T>` — returns a reference to the top item
- `fn pop(self) -> (Option<T>, Self)` — pops the top item, returns it and the rest

```rust
// TODO: Define Stack<T> and implement the methods

fn main() {
    let stack = Stack::new();
    let stack = stack.push(1);
    let stack = stack.push(2);
    let stack = stack.push(3);

    println!("Top: {:?}", stack.peek()); // Some(3)

    let (val, stack) = stack.pop();
    println!("Popped: {:?}", val);       // Some(3)
    println!("New top: {:?}", stack.peek()); // Some(2)
}
```

<details>
<summary>✅ Solution</summary>

```rust
enum Stack<T> {
    Empty,
    Entry(T, Box<Stack<T>>),
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack::Empty
    }

    fn push(self, item: T) -> Self {
        Stack::Entry(item, Box::new(self))
    }

    fn peek(&self) -> Option<&T> {
        match self {
            Stack::Empty => None,
            Stack::Entry(val, _) => Some(val),
        }
    }

    fn pop(self) -> (Option<T>, Self) {
        match self {
            Stack::Empty => (None, Stack::Empty),
            Stack::Entry(val, rest) => (Some(val), *rest),
        }
    }
}

fn main() {
    let stack = Stack::new();
    let stack = stack.push(1);
    let stack = stack.push(2);
    let stack = stack.push(3);

    println!("Top: {:?}", stack.peek()); // Some(3)

    let (val, stack) = stack.pop();
    println!("Popped: {:?}", val);       // Some(3)
    println!("New top: {:?}", stack.peek()); // Some(2)
}
```

</details>

---

### Exercise 3: Specialized Method — Magnitude for Numeric Points

Define a `Point<T>` struct with `x` and `y` fields. Implement:

- A generic `new(x: T, y: T) -> Self` constructor for all `T`
- A specialized `fn magnitude(&self) -> f64` method **only** for `Point<f64>`
- A specialized `fn manhattan(&self) -> i32` method **only** for `Point<i32>`

```rust
// TODO: Define Point<T> with generic + specialized impl blocks

fn main() {
    let pf = Point::new(3.0_f64, 4.0);
    println!("Magnitude: {}", pf.magnitude()); // 5.0
    // pf.manhattan(); // ERROR — not available for f64

    let pi = Point::new(3_i32, -4);
    println!("Manhattan: {}", pi.manhattan()); // 7
    // pi.magnitude(); // ERROR — not available for i32
}
```

<details>
<summary>✅ Solution</summary>

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point { x, y }
    }
}

impl Point<f64> {
    fn magnitude(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

impl Point<i32> {
    fn manhattan(&self) -> i32 {
        self.x.abs() + self.y.abs()
    }
}

fn main() {
    let pf = Point::new(3.0_f64, 4.0);
    println!("Magnitude: {}", pf.magnitude()); // 5.0

    let pi = Point::new(3_i32, -4);
    println!("Manhattan: {}", pi.manhattan()); // 7
}
```

</details>

---

## Summary

| Concept                    | Syntax / Example                              | Key Takeaway                                              |
|----------------------------|-----------------------------------------------|-----------------------------------------------------------|
| Generic struct             | `struct Point<T> { x: T, y: T }`             | One definition, works for any type                        |
| Multiple type params       | `struct Point<T, U> { x: T, y: U }`          | Each field can have a different type                      |
| Generic methods            | `impl<T> Point<T> { fn x(&self) -> &T }`     | Methods available for all `T`                             |
| Specialized methods        | `impl Point<f64> { fn dist(&self) -> f64 }`  | Methods only for specific concrete types                  |
| `Option<T>`                | `Some(T)` / `None`                            | You've been using generics all along                      |
| `Result<T, E>`             | `Ok(T)` / `Err(E)`                           | Two type parameters — success and error                   |
| Custom generic enums       | `enum Either<L, R> { Left(L), Right(R) }`    | Build your own parameterized data types                   |
| Recursive generic types    | `enum Tree<T> { Leaf(T), Node(Box<Tree<T>>..)}` | `Box` breaks the infinite-size recursion               |
| Monomorphization           | `Point<i32>` → 8 bytes, `Point<f64>` → 16    | Compiler generates concrete types — zero overhead         |
| Niche optimization         | `size_of::<Option<&T>>() == size_of::<&T>()`  | `None` reuses the null pointer — free `Option` wrapping  |
| Standard library generics  | `Vec<T>`, `HashMap<K,V>`, `Box<T>`, ...       | The entire standard library is built on generics          |
| Turbofish                  | `"42".parse::<i32>()`, `Vec::<i32>::new()`    | Explicit type parameters when inference isn't enough      |
| `PhantomData<T>`           | Zero-sized type-level marker                  | Use when `T` doesn't appear in fields                     |

### What's Next?

We can now **parameterize** structs and enums over types, but we haven't yet
constrained what those types can *do*. In the next tutorial, we'll learn how to
**define traits** — the contracts that describe shared behaviour across types.
Traits and generics together form the backbone of Rust's type system.

---

**Previous:** [← Generic Functions](./01-generic-functions.md) · **Next:** [Defining Traits →](./03-defining-traits.md)

<p align="center"><i>Tutorial 2 of 9 — Stage 8: Generics & Traits</i></p>

# Trait Bounds 🔒

> **"Trait bounds are the contract between generics and reality — they tell the compiler exactly what a type *can do*, so your generic code can actually do something useful."**

---

## Table of Contents

- [Why Trait Bounds?](#why-trait-bounds)
- [Inline Bounds](#inline-bounds)
- [Where Clauses](#where-clauses)
- [impl Trait Syntax](#impl-trait-syntax)
- [impl Trait in Return Position](#impl-trait-in-return-position)
- [Multiple Bounds](#multiple-bounds)
- [Conditional Method Implementation](#conditional-method-implementation)
- [Bound Propagation](#bound-propagation)
- [The Sized Bound](#the-sized-bound)
- [Trait Bounds in Other Languages](#trait-bounds-in-other-languages)
- [Advanced: Higher-Ranked Trait Bounds (HRTB)](#advanced-higher-ranked-trait-bounds-hrtb)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Why Trait Bounds?

A generic function like `fn do_something<T>(item: T)` accepts *any* type — but that
means the compiler knows *nothing* about `T`. You can't print it, clone it, compare it,
or call any methods on it. It's a blank slate.

Trait bounds fix this by saying: "I don't care *which* type you give me, as long as it
can do *this*."

```rust
use std::fmt::Display;

// Without a bound — won't compile!
// fn print_item<T>(item: T) {
//     println!("{}", item); // ❌ error: `T` doesn't implement `Display`
// }

// With a trait bound — compiles perfectly
fn print_item<T: Display>(item: T) {
    println!("{}", item); // ✅ T is guaranteed to implement Display
}

fn main() {
    print_item(42);           // i32 implements Display ✅
    print_item("hello");      // &str implements Display ✅
    print_item(3.14_f64);     // f64 implements Display ✅

    // print_item(vec![1, 2]); // ❌ Vec<i32> does NOT implement Display
}
```

The bound `T: Display` is a **constraint**. It narrows the infinite universe of possible
types down to only those that implement the `Display` trait. In exchange, your function
body gets to use everything `Display` provides — in this case, the ability to format
with `{}`.

Think of it as a deal:
- **Caller's side:** "I promise to give you a type that implements Display."
- **Function's side:** "Then I promise to only use Display methods on it."

This deal is enforced **entirely at compile time**. There is zero runtime cost.

---

## Inline Bounds

The most common syntax places the bound right after the type parameter in angle
brackets:

```rust
fn largest<T: PartialOrd>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}
```

You can require **multiple traits** on the same type parameter using `+`:

```rust
use std::fmt::{Display, Debug};

fn inspect<T: Display + Debug>(item: T) {
    println!("Display: {}", item);
    println!("Debug:   {:?}", item);
}

fn main() {
    inspect(42);        // i32 implements both Display and Debug ✅
    inspect("hello");   // &str implements both ✅
}
```

Multiple type parameters each get their own bounds:

```rust
use std::fmt::{Display, Debug};

fn compare_and_show<T: PartialOrd + Display, U: Debug>(a: T, b: T, label: U) {
    println!("{:?}: {} vs {} → winner: {}", label, a, b, if a >= b { a } else { b });
}
```

### When inline bounds get messy

Inline bounds work great for one or two simple constraints. But they start to crowd the
function signature when things get complex:

```rust
// This is getting hard to read...
fn process<T: Display + Clone + PartialOrd + Default, U: Debug + Into<String>, V: AsRef<str>>(
    a: T, b: U, c: V
) {
    // ...
}
```

When the signature gets unwieldy, reach for a `where` clause.

---

## Where Clauses

A `where` clause moves the bounds *after* the function signature, keeping the parameter
list clean:

```rust
use std::fmt::{Display, Debug};

fn process<T, U, V>(a: T, b: U, c: V)
where
    T: Display + Clone + PartialOrd + Default,
    U: Debug + Into<String>,
    V: AsRef<str>,
{
    println!("{}", a);
    println!("{:?}", b);
    println!("{}", c.as_ref());
}
```

Compare the readability — the function name, parameters, and return type are all visible
at a glance, with the constraints neatly organized below.

### Where clauses can express things inline bounds cannot

`where` clauses can constrain types that aren't type parameters themselves:

```rust
use std::fmt::Display;

// Constrain an associated type
fn print_first<I>(iter: I)
where
    I: Iterator,
    I::Item: Display,  // ← bound on an associated type, not a type parameter!
{
    if let Some(first) = iter.into_iter().next() {
        println!("First item: {}", first);
    }
}

fn main() {
    let nums = vec![10, 20, 30];
    print_first(nums.into_iter()); // i32 implements Display ✅
}
```

You can also write bounds on *concrete* types (rarely needed, but legal):

```rust
fn foo<T>(x: T)
where
    String: From<T>,  // bound on String, not T!
{
    let s = String::from(x);
    println!("{}", s);
}
```

### When to use which

| Situation | Preferred Style |
|---|---|
| One type parameter, one or two bounds | Inline: `fn foo<T: Clone>(x: T)` |
| Multiple type parameters, each with bounds | `where` clause |
| Bounds on associated types | `where` clause (required) |
| Team style guide says so | Follow the guide |

Both are semantically identical — it's purely a readability choice.

---

## impl Trait Syntax

Rust provides `impl Trait` as syntactic sugar for simple single-parameter bounds in
**argument position**:

```rust
use std::fmt::Display;

// These two are equivalent:
fn print_a<T: Display>(item: T) {
    println!("{}", item);
}

fn print_b(item: &impl Display) {
    println!("{}", item);
}

fn main() {
    print_a(&42);
    print_b(&42);  // same effect
}
```

`impl Display` in argument position means "some type that implements Display." The
compiler still monomorphizes — there's no dynamic dispatch.

### With multiple bounds

You can use `+` with `impl Trait` too:

```rust
use std::fmt::{Display, Debug};

fn log(item: &(impl Display + Debug)) {
    println!("Display: {}", item);
    println!("Debug:   {:?}", item);
}
```

### Limitations of `impl Trait` in argument position

You **cannot** use `impl Trait` when you need to name the type elsewhere:

```rust
use std::fmt::Display;

// ❌ Can't do this — how do you say "both parameters are the SAME type"?
// fn same_type(a: impl Display, b: impl Display) { ... }
// ↑ These could be two DIFFERENT types!

// ✅ Use a named type parameter instead
fn same_type<T: Display>(a: T, b: T) {
    println!("{} and {}", a, b);
}
```

With `impl Trait`, each occurrence introduces a *separate* anonymous type parameter. If
you need two parameters to be the *same* type, you must name the type parameter
explicitly.

### When to use `impl Trait` in argument position

| Use `impl Trait` when... | Use named generics when... |
|---|---|
| Single occurrence of the type | Multiple parameters share the same type |
| You don't need to reference the type elsewhere | The type appears in `where` clauses |
| Simple, readable signatures | Complex multi-parameter bounds |

---

## impl Trait in Return Position

`impl Trait` in return position means "I'm returning *some* concrete type that
implements this trait, but I'm not telling you which one":

```rust
use std::fmt::Display;

fn greeting(name: &str) -> impl Display {
    format!("Hello, {}!", name)
    // Returns a String, but the caller only knows it implements Display
}

fn main() {
    let msg = greeting("Rustacean");
    println!("{}", msg);  // ✅ can use Display methods

    // But you can't do String-specific things:
    // msg.push_str("!!");  // ❌ the type is opaque
}
```

### Why this matters: closures and iterators

Closures have **unnameable** types. Without `impl Trait`, you'd need `Box<dyn Fn(...)>`:

```rust
// Each closure has a unique, anonymous type — you can't write it explicitly
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |y| x + y
}

fn main() {
    let add_five = make_adder(5);
    println!("{}", add_five(10));  // 15
}
```

Iterator adapters produce deeply nested unnameable types. `impl Iterator` saves the day:

```rust
fn even_squares(limit: i32) -> impl Iterator<Item = i32> {
    (1..limit)
        .filter(|x| x % 2 == 0)
        .map(|x| x * x)
    // The actual type is something like:
    // Map<Filter<Range<i32>, [closure]>, [closure]>
    // Good luck writing that out!
}

fn main() {
    for n in even_squares(10) {
        println!("{}", n);  // 4, 16, 36, 64
    }
}
```

### Important rule: one concrete type per return

A function returning `impl Trait` must return the **same concrete type** from all return
paths:

```rust
use std::fmt::Display;

// ❌ Won't compile — returns different types
// fn either(flag: bool) -> impl Display {
//     if flag {
//         42          // i32
//     } else {
//         "hello"     // &str
//     }
// }

// ✅ To return different types, use a trait object (Box<dyn Trait>)
fn either(flag: bool) -> Box<dyn Display> {
    if flag {
        Box::new(42)
    } else {
        Box::new("hello")
    }
}
```

This restriction exists because `impl Trait` is **static dispatch** — the compiler must
know the concrete type at compile time to monomorphize. Trait objects (`dyn Trait`) use
**dynamic dispatch** for runtime polymorphism.

---

## Multiple Bounds

When a type parameter needs to satisfy many traits, list them all with `+`:

```rust
use std::fmt::{Display, Debug};
use std::hash::Hash;

fn catalog_item<T>(item: T)
where
    T: Display + Debug + Clone + PartialEq + Hash,
{
    println!("Display: {}", item);
    println!("Debug:   {:?}", item);

    let copy = item.clone();
    assert_eq!(item, copy);

    // Can use in HashMaps, HashSets, etc.
}

fn main() {
    catalog_item(String::from("widget"));
    catalog_item(42_i32);
}
```

Every bound joined by `+` is an **AND** — the type must implement *all* of them. There
is no "OR" combinator for trait bounds (you'd use an enum or trait object for that).

### Supertraits vs. multiple bounds

Don't confuse multiple bounds with **supertraits**. Multiple bounds constrain a *type
parameter*, while supertraits constrain a *trait definition*:

```rust
use std::fmt::Display;

// Supertrait: anything implementing PrettyPrint MUST also implement Display
trait PrettyPrint: Display {
    fn pretty(&self) -> String;
}

// Multiple bounds: T must implement both traits independently
fn show<T: Display + Clone>(item: T) {
    let copy = item.clone();
    println!("{}", copy);
}
```

Both mechanisms ensure the required traits are available, but supertraits bake the
requirement into the trait itself, while multiple bounds are per-function.

---

## Conditional Method Implementation

One of Rust's most powerful features: you can implement methods on a generic struct
**only when** the type parameter meets certain bounds.

```rust
use std::fmt::Display;

struct Wrapper<T> {
    value: T,
}

// These methods are available for ALL Wrapper<T>
impl<T> Wrapper<T> {
    fn new(value: T) -> Self {
        Wrapper { value }
    }

    fn into_inner(self) -> T {
        self.value
    }
}

// These methods are ONLY available when T: Display
impl<T: Display> Wrapper<T> {
    fn show(&self) {
        println!("Wrapped value: {}", self.value);
    }
}

// These methods are ONLY available when T: Display + PartialOrd
impl<T: Display + PartialOrd> Wrapper<T> {
    fn max_display(a: &Self, b: &Self) {
        if a.value >= b.value {
            println!("Max: {}", a.value);
        } else {
            println!("Max: {}", b.value);
        }
    }
}

fn main() {
    let w1 = Wrapper::new(42);
    w1.show();  // ✅ i32 implements Display

    let w2 = Wrapper::new(vec![1, 2, 3]);
    // w2.show();  // ❌ Vec<i32> doesn't implement Display
    // But w2.into_inner() still works!
    let v = w2.into_inner();
    println!("{:?}", v);
}
```

### Blanket implementations

The standard library uses this pattern extensively. For example, `ToString` is
implemented for **every type that implements Display**:

```rust
// In the standard library (simplified):
impl<T: Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{}", self)
    }
}
```

This is called a **blanket implementation** — it implements a trait for all types
matching a bound, not just one specific type. That's why you can call `.to_string()` on
any `Display` type without implementing `ToString` yourself.

Another classic blanket impl:

```rust
// Also in std (simplified):
// Every reference to a Display type is itself Display
// impl<T: Display + ?Sized> Display for &T { ... }
```

---

## Bound Propagation

When your generic function calls *another* generic function, you must **propagate** the
bounds — your function's type parameter needs at least the same constraints as the
inner function requires.

```rust
use std::fmt::Display;

fn inner<T: Display>(item: &T) {
    println!("inner: {}", item);
}

// ❌ This won't compile — T has no bounds, but inner() requires Display
// fn outer<T>(item: &T) {
//     inner(item);  // error[E0277]: `T` doesn't implement `Display`
// }

// ✅ Propagate the bound
fn outer<T: Display>(item: &T) {
    inner(item);  // now T: Display, so inner() is satisfied
}

fn main() {
    outer(&"works!");
}
```

### The cascade effect

Bound propagation cascades through your call stack. If function A calls B calls C, and C
requires `T: Clone + Debug`, then B needs at least `T: Clone + Debug`, and A needs at
least `T: Clone + Debug`:

```rust
use std::fmt::Debug;

fn level_c<T: Clone + Debug>(item: &T) {
    let copy = item.clone();
    println!("C: {:?}", copy);
}

fn level_b<T: Clone + Debug>(item: &T) {
    println!("B: entering");
    level_c(item);  // needs Clone + Debug → propagated
}

fn level_a<T: Clone + Debug>(item: &T) {
    println!("A: entering");
    level_b(item);  // needs Clone + Debug → propagated
}

fn main() {
    level_a(&42);
}
```

### Common error message

When you forget to propagate a bound, you'll see:

```
error[E0277]: the trait bound `T: Display` is not satisfied
  --> src/main.rs:X:Y
   |
   |     inner(item);
   |           ^^^^ the trait `Display` is not implemented for `T`
   |
help: consider restricting type parameter `T`
   |
   | fn outer<T: std::fmt::Display>(item: &T) {
   |           +++++++++++++++++++
```

The compiler's suggestion is almost always exactly what you need — add the missing bound
to your function's type parameter. Read these messages carefully; Rust's error
diagnostics are excellent.

---

## The Sized Bound

Every type parameter in Rust has an **implicit bound**: `T: Sized`. This means the
compiler assumes it knows the size of `T` at compile time.

```rust
// These two signatures are equivalent:
fn foo<T>(x: T) { }
fn bar<T: Sized>(x: T) { }  // explicit, but redundant
```

### Why Sized by default?

To pass a value on the stack, the compiler needs to know its size. Most types are `Sized`
— `i32` is 4 bytes, `String` is 24 bytes (pointer + length + capacity), etc.

But some types are **not** `Sized` (called *dynamically sized types* or DSTs):

| Type | Why it's unsized |
|---|---|
| `str` | String slice — length unknown at compile time |
| `[T]` | Slice — how many elements? |
| `dyn Trait` | Could be any type implementing the trait |

You always interact with these behind a reference or pointer: `&str`, `&[T]`,
`Box<dyn Trait>`.

### Opting out with `?Sized`

To write a generic function that accepts unsized types, use the `?Sized` bound:

```rust
use std::fmt::Display;

// Only accepts Sized types (the default)
fn sized_only<T: Display>(item: &T) {
    println!("{}", item);
}

// Accepts both Sized AND unsized types
fn any_size<T: Display + ?Sized>(item: &T) {
    println!("{}", item);
}

fn main() {
    let s: &str = "hello";        // str is unsized
    // sized_only(s);             // ❌ str is not Sized
    sized_only(&String::from("hello"));  // ✅ String is Sized

    any_size(s);                   // ✅ works with unsized str
    any_size(&42);                 // ✅ also works with Sized i32
}
```

The `?` in `?Sized` means "maybe Sized" — it *relaxes* the implicit `Sized` bound rather
than adding a new constraint. It's the only `?Trait` syntax in Rust.

### When to use `?Sized`

Use `?Sized` when your function only accesses `T` through a reference and you want
maximum flexibility. The standard library does this often:

```rust
// std::borrow::Borrow (simplified)
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}

// This lets you write functions that work with both String and str,
// both Vec<T> and [T], etc.
fn print_borrowed<T: std::borrow::Borrow<str>>(item: T) {
    println!("{}", item.borrow());
}
```

If you only ever take `&T` (never own `T`), consider adding `?Sized` to your bounds. It
costs nothing and makes your API more flexible.

---

## Trait Bounds in Other Languages

Rust's trait bounds aren't a novelty — many languages have mechanisms to constrain
generic types. Here's how they compare:

### Java — Bounded Type Parameters

```java
// Upper bound: T must extend Comparable
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

// Multiple bounds use &
public static <T extends Comparable<T> & Serializable> void process(T item) {
    // T must implement both Comparable and Serializable
}
```

Java uses `extends` for both class inheritance and interface bounds. Only one class bound
is allowed, but multiple interface bounds can follow with `&`.

### C# — Constraint Clauses

```csharp
// where clause, similar to Rust's
public static T Max<T>(T a, T b) where T : IComparable<T>
{
    return a.CompareTo(b) >= 0 ? a : b;
}

// Multiple constraints
public static void Process<T>(T item) where T : IComparable<T>, IDisposable
{
    // T must implement both interfaces
}
```

C#'s `where` clause looks very similar to Rust's. It also supports `where T : struct`
(value type) and `where T : class` (reference type), which have no Rust equivalent.

### C++ — Concepts (C++20)

```cpp
#include <concepts>

// Before C++20: templates had no bounds — errors were incomprehensible
// After C++20: concepts provide trait-bound-like constraints

template<typename T>
concept Displayable = requires(T t) {
    { std::cout << t } -> std::same_as<std::ostream&>;
};

template<Displayable T>
void print(T item) {
    std::cout << item << std::endl;
}
```

C++ templates were unconstrained for decades. Concepts, standardized in C++20 after
nearly 20 years of proposals, finally brought compile-time constraint checking. Before
concepts, template errors produced famously unreadable error messages.

### Go — Type Constraints (Go 1.18+)

```go
// Type constraint using interface
func Max[T constraints.Ordered](a, b T) T {
    if a >= b {
        return a
    }
    return b
}

// Custom constraint
type Number interface {
    ~int | ~float64  // union of underlying types
}
```

Go's generics (added in 1.18) use interfaces as type constraints. The `comparable`
built-in constraint allows `==` and `!=`.

### Haskell — Typeclass Constraints

```haskell
-- Typeclass constraints before =>
maxOf :: (Ord a) => a -> a -> a
maxOf x y = if x >= y then x else y

-- Multiple constraints in a tuple
inspect :: (Show a, Eq a) => a -> String
inspect x = show x
```

Haskell's typeclass system *directly inspired* Rust's traits. The syntax
`(Show a, Eq a) =>` maps almost 1:1 to Rust's `T: Display + PartialEq`.

### Swift — Protocol Constraints

```swift
// Protocol constraint with :
func max<T: Comparable>(_ a: T, _ b: T) -> T {
    return a >= b ? a : b
}

// Multiple constraints with &
func process<T: Equatable & Hashable>(_ item: T) {
    // T must conform to both protocols
}
```

Swift's protocol constraints use `:` for single bounds and `&` for multiple, closely
resembling Rust's syntax.

### Comparison Table

| Feature | Rust | Java | C# | C++ | Go | Haskell | Swift |
|---|---|---|---|---|---|---|---|
| Single bound | `T: Trait` | `T extends I` | `where T : I` | `concept` | `[T I]` | `(C a) =>` | `T: P` |
| Multiple bounds | `T: A + B` | `T extends A & B` | `T : A, B` | `concept` | `interface{}` | `(A a, B a)` | `T: A & B` |
| Where clause | `where T: A` | N/A | `where T : A` | `requires` | N/A | Context | `where T: A` |
| Return bound | `-> impl T` | Wildcard `?` | N/A | `auto` | N/A | Inferred | `some P` |
| Opt-out bound | `?Sized` | N/A | N/A | N/A | N/A | N/A | N/A |
| Zero-cost | ✅ monomorphization | ❌ type erasure | ✅ value types | ✅ templates | ❌ boxing | ❌ dictionary | ✅ specialization |
| Compile-time checked | ✅ | ✅ | ✅ | ✅ (C++20) | ✅ | ✅ | ✅ |

Rust's system is most comparable to Haskell's typeclasses and Swift's protocol
constraints, with the unique addition of `?Sized` and full monomorphization guarantees.

---

## Advanced: Higher-Ranked Trait Bounds (HRTB)

Sometimes you need a bound that works **for all possible lifetimes**, not just one
specific lifetime. This is where *higher-ranked trait bounds* come in.

```rust
// A function that takes a closure which works with ANY lifetime
fn apply_to_str<F>(f: F, s: &str) -> String
where
    F: for<'a> Fn(&'a str) -> &'a str,  // ← HRTB
{
    f(s).to_string()
}

fn main() {
    let trimmer = |s: &str| s.trim();
    let result = apply_to_str(trimmer, "  hello  ");
    println!("'{}'", result);  // 'hello'
}
```

The `for<'a> Fn(&'a str) -> &'a str` syntax means: "F must implement `Fn(&str) -> &str`
for **every possible lifetime** `'a`." It's called "higher-ranked" because the `for<'a>`
quantifier is at a higher level than the usual function-level lifetime parameters.

### When you encounter HRTB

Most Rust programmers rarely write HRTB explicitly. The compiler inserts them
automatically in common cases:

```rust
// You write this:
fn takes_closure<F: Fn(&str) -> &str>(f: F) { }

// The compiler desugars it to:
// fn takes_closure<F: for<'a> Fn(&'a str) -> &'a str>(f: F) { }
```

You'll encounter HRTB explicitly when:
- Working with complex closure types in trait definitions
- Writing generic parsers or deserializers (e.g., Serde)
- Implementing async traits with borrowed data

> **Note:** HRTBs and their interaction with lifetimes will be covered in depth in
> **Stage 9: Lifetimes**. For now, just know this syntax exists and trust the compiler's
> automatic desugaring for closures.

---

## Exercises

### Exercise 1: Generic Minimum with Bounds

Write a function `minimum` that returns the smaller of two values. It should work with
any type that can be compared and cloned.

```rust
use std::fmt::Display;

// TODO: Add the correct trait bounds
fn minimum<T: ???>(a: T, b: T) -> T {
    if a <= b { a } else { b }
}

fn main() {
    println!("{}", minimum(10, 20));              // 10
    println!("{}", minimum("apple", "banana"));   // apple
    println!("{}", minimum(3.14, 2.72));          // 2.72
}
```

<details>
<summary>✅ Solution</summary>

```rust
fn minimum<T: PartialOrd>(a: T, b: T) -> T {
    if a <= b { a } else { b }
}

fn main() {
    println!("{}", minimum(10, 20));              // 10
    println!("{}", minimum("apple", "banana"));   // apple
    println!("{}", minimum(3.14, 2.72));          // 2.72
}
```

`PartialOrd` is the only bound needed — it provides `<=`. We don't need `Clone` because
we're moving values, not copying. We don't need `Display` for the function itself;
`println!` in `main` resolves the bound on its own.

</details>

### Exercise 2: Conditional Methods on a Container

Create a `Bag<T>` struct that holds a `Vec<T>`. Implement:
- `new()` and `add()` for **all** `T`
- `summary()` that prints all items — only when `T: Display`
- `sort_contents()` that sorts in place — only when `T: Ord`
- `deduplicate()` that removes consecutive duplicates — only when `T: PartialEq`

```rust
use std::fmt::Display;

struct Bag<T> {
    items: Vec<T>,
}

// TODO: Implement the four impl blocks with appropriate bounds

fn main() {
    let mut bag = Bag::new();
    bag.add(3);
    bag.add(1);
    bag.add(2);
    bag.add(2);

    bag.summary();          // should print: [3, 1, 2, 2]
    bag.sort_contents();
    bag.summary();          // should print: [1, 2, 2, 3]
    bag.deduplicate();
    bag.summary();          // should print: [1, 2, 3]
}
```

<details>
<summary>✅ Solution</summary>

```rust
use std::fmt::Display;

struct Bag<T> {
    items: Vec<T>,
}

// Available for ALL T
impl<T> Bag<T> {
    fn new() -> Self {
        Bag { items: Vec::new() }
    }

    fn add(&mut self, item: T) {
        self.items.push(item);
    }
}

// Only when T: Display
impl<T: Display> Bag<T> {
    fn summary(&self) {
        let items: Vec<String> = self.items.iter().map(|i| i.to_string()).collect();
        println!("[{}]", items.join(", "));
    }
}

// Only when T: Ord
impl<T: Ord> Bag<T> {
    fn sort_contents(&mut self) {
        self.items.sort();
    }
}

// Only when T: PartialEq
impl<T: PartialEq> Bag<T> {
    fn deduplicate(&mut self) {
        self.items.dedup();
    }
}

fn main() {
    let mut bag = Bag::new();
    bag.add(3);
    bag.add(1);
    bag.add(2);
    bag.add(2);

    bag.summary();          // [3, 1, 2, 2]
    bag.sort_contents();
    bag.summary();          // [1, 2, 2, 3]
    bag.deduplicate();
    bag.summary();          // [1, 2, 3]
}
```

Each `impl` block has its own bounds. `i32` satisfies all of them, so all methods are
available. If you put a type that doesn't implement `Ord` (like `f64`), the
`sort_contents()` method simply wouldn't exist for that `Bag<f64>`.

</details>

### Exercise 3: Return `impl Trait`

Write a function `make_repeater` that takes a `String` and a count, and returns an
iterator that yields that string `count` times. Use `impl Iterator<Item = String>` as
the return type.

```rust
// TODO: implement make_repeater
fn make_repeater(text: String, count: usize) -> impl Iterator<Item = String> {
    todo!()
}

fn main() {
    for line in make_repeater(String::from("echo"), 3) {
        println!("{}", line);
    }
    // Should print:
    // echo
    // echo
    // echo
}
```

<details>
<summary>✅ Solution</summary>

```rust
fn make_repeater(text: String, count: usize) -> impl Iterator<Item = String> {
    std::iter::repeat(text).take(count)
}

fn main() {
    for line in make_repeater(String::from("echo"), 3) {
        println!("{}", line);
    }
    // echo
    // echo
    // echo
}
```

`std::iter::repeat(text)` creates an infinite iterator, and `.take(count)` limits it.
The return type `impl Iterator<Item = String>` hides the concrete `Take<Repeat<String>>`
type from the caller.

An alternative using `(0..count).map(...)`:

```rust
fn make_repeater(text: String, count: usize) -> impl Iterator<Item = String> {
    (0..count).map(move |_| text.clone())
}
```

This version clones the string each iteration. The `move` keyword captures `text` into
the closure.

</details>

---

## Summary

| Concept | Syntax | Purpose |
|---|---|---|
| Inline bound | `fn foo<T: Trait>(x: T)` | Simple, common constraints |
| Multiple bounds | `T: A + B + C` | Require several traits (AND logic) |
| Where clause | `where T: A + B` | Complex/readable constraints |
| `impl Trait` argument | `fn foo(x: &impl Trait)` | Sugar for single-use type params |
| `impl Trait` return | `fn foo() -> impl Trait` | Opaque return types (closures, iterators) |
| Conditional methods | `impl<T: Trait> S<T> { }` | Methods gated on type capabilities |
| Blanket impl | `impl<T: A> B for T` | Implement a trait for all qualifying types |
| Bound propagation | Add bounds to outer functions | Forward requirements up the call stack |
| `?Sized` | `T: ?Sized` | Accept dynamically sized types behind refs |
| HRTB | `for<'a> Fn(&'a T)` | Lifetime-generic bounds (advanced) |

**Key takeaways:**
- Trait bounds constrain generic types so your code can *actually use* them.
- `+` means AND — the type must satisfy **all** listed bounds.
- `where` clauses and inline bounds are semantically identical; choose for readability.
- `impl Trait` in return position is essential for closures and iterator adaptors.
- Conditional `impl` blocks let you add methods only when the type parameter qualifies.
- `?Sized` relaxes the implicit `Sized` bound for maximum API flexibility.
- When in doubt, read the compiler error — it almost always tells you which bound is
  missing and where to add it.

---

**Previous:** [← Implementing Traits](./04-implementing-traits.md) · **Next:** [Common Std Traits →](./06-common-std-traits.md)

<p align="center"><i>Tutorial 5 of 9 — Stage 8: Generics & Traits</i></p>

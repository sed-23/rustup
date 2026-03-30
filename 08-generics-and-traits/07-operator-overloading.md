# Operator Overloading 🔧

> **In Rust, every operator is a trait method in disguise. Implement the trait, and your types speak the language of `+`, `-`, `*`, `==`, and `[]`.**

---

## Table of Contents

- [What Is Operator Overloading?](#what-is-operator-overloading)
- [The Add Trait — Implementing +](#the-add-trait--implementing-)
- [All Arithmetic Operator Traits](#all-arithmetic-operator-traits)
- [Compound Assignment Operators](#compound-assignment-operators)
- [Comparison Operators](#comparison-operators)
- [Index and IndexMut](#index-and-indexmut)
- [Operator Overloading with Different Types](#operator-overloading-with-different-types)
- [Operator Overloading in Other Languages](#operator-overloading-in-other-languages)
- [Real-World Examples](#real-world-examples)
- [Guidelines — When to Overload](#guidelines--when-to-overload)
- [Exercises](#exercises)
- [Summary](#summary)

---

## What Is Operator Overloading?

When you write `2 + 3`, the compiler knows how to add integers. But what about this?

```rust
let a = Point { x: 1.0, y: 2.0 };
let b = Point { x: 3.0, y: 4.0 };
let c = a + b;  // Can we do this?
```

Out of the box — **no**. Rust doesn't know how to add two `Point` values. But Rust lets you *teach* it.

### Operators Are Trait Method Calls

This is the key insight: **in Rust, every operator maps to a trait in `std::ops`** (or `std::cmp` for comparisons). When you write `a + b`, the compiler actually calls:

```rust
a.add(b)   // std::ops::Add::add(a, b)
```

Here's the mapping:

| Operator | Trait | Method |
|----------|-------|--------|
| `a + b` | `Add` | `a.add(b)` |
| `a - b` | `Sub` | `a.sub(b)` |
| `a * b` | `Mul` | `a.mul(b)` |
| `a / b` | `Div` | `a.div(b)` |
| `a % b` | `Rem` | `a.rem(b)` |
| `-a` | `Neg` | `a.neg()` |
| `a == b` | `PartialEq` | `a.eq(&b)` |
| `a < b` | `PartialOrd` | `a.partial_cmp(&b)` |
| `a[i]` | `Index` | `a.index(i)` |
| `a[i] = v` | `IndexMut` | `*a.index_mut(i) = v` |
| `a += b` | `AddAssign` | `a.add_assign(b)` |

So "operator overloading" in Rust really means: **implement the corresponding trait**.

---

## The Add Trait — Implementing +

Let's start with the most common operator: addition.

### The Trait Definition

Here's what `std::ops::Add` looks like in the standard library:

```rust
pub trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

Three things to notice:

1. **`Rhs = Self`** — The right-hand side defaults to the same type as the left. So `impl Add for Point` means "Point + Point".
2. **`type Output`** — An associated type that defines the return type. Adding two `Point`s gives a `Point`, but adding two `&str`s gives a `String`.
3. **`self`, not `&self`** — The `add` method **consumes** both operands by value. This is a deliberate design choice we'll discuss below.

### Full Example: Point + Point

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    let a = Point { x: 1.0, y: 2.0 };
    let b = Point { x: 3.0, y: 4.0 };

    let c = a + b;
    println!("{:?}", c);  // Point { x: 4.0, y: 6.0 }

    // Because Point is Copy, a and b are still valid:
    println!("a = {:?}, b = {:?}", a, b);
}
```

### Why Does `add` Take `self` by Value?

You might wonder: why does `+` consume its operands instead of borrowing them?

**Efficiency.** Consider a type like `String`:

```rust
let greeting = String::from("Hello, ") + &world;
```

When two `String`s are concatenated, Rust can **reuse the left operand's heap allocation** by appending to its buffer in-place, instead of allocating a brand-new buffer. If `add` took `&self`, it would always have to clone.

For `Copy` types like our `Point`, this doesn't matter — they're just copied. But for heap-allocated types, this design choice enables zero-copy optimization.

> **Rule of thumb:** If your type is small and `Copy`, operator overloading is painless. If your type owns heap data, think about whether consuming the left operand makes sense.

---

## All Arithmetic Operator Traits

All arithmetic operator traits live in `std::ops`. They all follow the same pattern as `Add`:

### Binary Arithmetic

```rust
use std::ops::{Add, Sub, Mul, Div, Rem};

#[derive(Debug, Clone, Copy)]
struct Vec2 {
    x: f64,
    y: f64,
}

impl Add for Vec2 {
    type Output = Vec2;
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

impl Sub for Vec2 {
    type Output = Vec2;
    fn sub(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x - rhs.x, y: self.y - rhs.y }
    }
}

impl Mul for Vec2 {
    type Output = f64;  // Dot product returns a scalar!
    fn mul(self, rhs: Vec2) -> f64 {
        self.x * rhs.x + self.y * rhs.y
    }
}

impl Div for Vec2 {
    type Output = Vec2;
    fn div(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x / rhs.x, y: self.y / rhs.y }
    }
}

impl Rem for Vec2 {
    type Output = Vec2;
    fn rem(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x % rhs.x, y: self.y % rhs.y }
    }
}
```

### Unary Negation

`Neg` is different — it's a unary operator with no right-hand side:

```rust
use std::ops::Neg;

impl Neg for Vec2 {
    type Output = Vec2;
    fn neg(self) -> Vec2 {
        Vec2 { x: -self.x, y: -self.y }
    }
}

fn main() {
    let v = Vec2 { x: 3.0, y: -4.0 };
    let neg_v = -v;
    println!("{:?}", neg_v);  // Vec2 { x: -3.0, y: 4.0 }
}
```

### Bitwise Operators

Rust also has traits for bitwise operators:

| Operator | Trait | Description |
|----------|-------|-------------|
| `a & b` | `BitAnd` | Bitwise AND |
| `a \| b` | `BitOr` | Bitwise OR |
| `a ^ b` | `BitXor` | Bitwise XOR |
| `!a` | `Not` | Bitwise NOT |
| `a << n` | `Shl` | Shift left |
| `a >> n` | `Shr` | Shift right |

These are useful for custom bit-flag types, permission sets, or low-level data structures.

---

## Compound Assignment Operators

Operators like `+=`, `-=`, `*=` are separate traits that take `&mut self`:

```rust
pub trait AddAssign<Rhs = Self> {
    fn add_assign(&mut self, rhs: Rhs);
}
```

Notice: **no `Output` type**. These modify in place and return nothing.

### Example: Implementing += for Vec2

```rust
use std::ops::AddAssign;

impl AddAssign for Vec2 {
    fn add_assign(&mut self, rhs: Vec2) {
        self.x += rhs.x;
        self.y += rhs.y;
    }
}

fn main() {
    let mut position = Vec2 { x: 0.0, y: 0.0 };
    let velocity = Vec2 { x: 1.5, y: 2.5 };

    position += velocity;
    println!("{:?}", position);  // Vec2 { x: 1.5, y: 2.5 }

    position += velocity;
    println!("{:?}", position);  // Vec2 { x: 3.0, y: 5.0 }
}
```

### All Compound Assignment Traits

| Operator | Trait |
|----------|-------|
| `+=` | `AddAssign` |
| `-=` | `SubAssign` |
| `*=` | `MulAssign` |
| `/=` | `DivAssign` |
| `%=` | `RemAssign` |
| `&=` | `BitAndAssign` |
| `\|=` | `BitOrAssign` |
| `^=` | `BitXorAssign` |
| `<<=` | `ShlAssign` |
| `>>=` | `ShrAssign` |

> **Important:** Implementing `Add` does NOT automatically give you `AddAssign`. You must implement them separately. This gives you full control — maybe `+=` should mutate in place for efficiency while `+` returns a new value.

---

## Comparison Operators

Comparison operators use traits from `std::cmp`, not `std::ops`:

### PartialEq — == and !=

```rust
pub trait PartialEq<Rhs = Self> {
    fn eq(&self, other: &Rhs) -> bool;
    fn ne(&self, other: &Rhs) -> bool { !self.eq(other) }  // default impl
}
```

Unlike arithmetic operators, comparisons take `&self` — comparing values shouldn't consume them!

```rust
#[derive(Debug)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl PartialEq for Color {
    fn eq(&self, other: &Color) -> bool {
        self.r == other.r && self.g == other.g && self.b == other.b
    }
}

fn main() {
    let red = Color { r: 255, g: 0, b: 0 };
    let also_red = Color { r: 255, g: 0, b: 0 };
    let blue = Color { r: 0, g: 0, b: 255 };

    println!("{}", red == also_red);  // true
    println!("{}", red != blue);      // true  (uses default ne())
}
```

> **Tip:** For most types, just `#[derive(PartialEq)]` and let the compiler generate the implementation.

### PartialOrd — <, >, <=, >=

```rust
pub trait PartialOrd<Rhs = Self>: PartialEq<Rhs> {
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;
}
```

Notice the supertrait bound: you must implement `PartialEq` before `PartialOrd`.

```rust
use std::cmp::Ordering;

#[derive(Debug, PartialEq)]
struct Score(u32);

impl PartialOrd for Score {
    fn partial_cmp(&self, other: &Score) -> Option<Ordering> {
        self.0.partial_cmp(&other.0)
    }
}

fn main() {
    let high = Score(100);
    let low = Score(50);

    println!("{}", high > low);   // true
    println!("{}", low >= high);  // false
}
```

We covered `PartialEq`, `Eq`, `PartialOrd`, and `Ord` in depth in [Tutorial 6: Common Std Traits](./06-common-std-traits.md). Here we're just showing them in the operator overloading context.

---

## Index and IndexMut

The `Index` and `IndexMut` traits let you use the `[]` subscript operator on your types.

### The Index Trait

```rust
pub trait Index<Idx> {
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}
```

Key points:
- `Idx` is the type used for indexing (e.g., `usize`, a `Range`, or even a `String`)
- `Output` is the type you get back (with `?Sized` allowing unsized types like `str` or `[T]`)
- Returns a **reference** — you're borrowing from the container

### Example: A Custom Matrix Type

```rust
use std::ops::Index;

struct Matrix {
    data: Vec<Vec<f64>>,
    rows: usize,
    cols: usize,
}

impl Matrix {
    fn new(rows: usize, cols: usize) -> Self {
        Matrix {
            data: vec![vec![0.0; cols]; rows],
            rows,
            cols,
        }
    }
}

impl Index<usize> for Matrix {
    type Output = Vec<f64>;

    fn index(&self, row: usize) -> &Vec<f64> {
        &self.data[row]
    }
}

fn main() {
    let mat = Matrix::new(3, 3);
    println!("{:?}", mat[0]);     // [0.0, 0.0, 0.0]
    println!("{}", mat[1][2]);    // 0.0  — double indexing works!
}
```

### Indexing with a Tuple — `mat[(row, col)]`

You can index with *any* type. Let's use a tuple for 2D access:

```rust
use std::ops::Index;

impl Index<(usize, usize)> for Matrix {
    type Output = f64;

    fn index(&self, (row, col): (usize, usize)) -> &f64 {
        &self.data[row][col]
    }
}

fn main() {
    let mat = Matrix::new(3, 3);
    println!("{}", mat[(1, 2)]);  // 0.0 — nice, clean syntax!
}
```

### IndexMut — Mutable Subscript

```rust
use std::ops::IndexMut;

impl IndexMut<(usize, usize)> for Matrix {
    fn index_mut(&mut self, (row, col): (usize, usize)) -> &mut f64 {
        &mut self.data[row][col]
    }
}

fn main() {
    let mut mat = Matrix::new(3, 3);
    mat[(0, 0)] = 1.0;
    mat[(1, 1)] = 1.0;
    mat[(2, 2)] = 1.0;
    // Now it's an identity matrix!
    println!("{}", mat[(0, 0)]);  // 1.0
}
```

> **Note:** `IndexMut` requires `Index` as a supertrait — you must implement `Index` first.

---

## Operator Overloading with Different Types

So far, we've been adding `Point + Point`. But what about `Point + f64` or `f64 * Point`?

### Adding a Scalar to a Point

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

// Point + Point (same type)
impl Add for Point {
    type Output = Point;
    fn add(self, other: Point) -> Point {
        Point { x: self.x + other.x, y: self.y + other.y }
    }
}

// Point + f64 (add scalar to both components)
impl Add<f64> for Point {
    type Output = Point;
    fn add(self, scalar: f64) -> Point {
        Point { x: self.x + scalar, y: self.y + scalar }
    }
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };

    let q = p + Point { x: 3.0, y: 4.0 };
    println!("{:?}", q);  // Point { x: 4.0, y: 6.0 }

    let r = p + 10.0;
    println!("{:?}", r);  // Point { x: 11.0, y: 12.0 }
}
```

### Implementing `f64 * Point` — The Reverse Direction

What if we want `2.0 * point`? Now the left-hand side is `f64`, not `Point`:

```rust
use std::ops::Mul;

// Point * f64 — scale a point
impl Mul<f64> for Point {
    type Output = Point;
    fn mul(self, scalar: f64) -> Point {
        Point { x: self.x * scalar, y: self.y * scalar }
    }
}

// f64 * Point — scale a point (reverse order)
impl Mul<Point> for f64 {
    type Output = Point;
    fn mul(self, point: Point) -> Point {
        Point { x: self * point.x, y: self * point.y }
    }
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };

    let a = p * 3.0;       // Point * f64
    let b = 3.0 * p;       // f64 * Point

    println!("{:?}", a);  // Point { x: 3.0, y: 6.0 }
    println!("{:?}", b);  // Point { x: 3.0, y: 6.0 }
}
```

The key insight: **the left-hand operand is `self`, and the right-hand operand is `rhs`**. So `f64 * Point` means `impl Mul<Point> for f64`.

### Generic Operator Implementations

You can even write generic operator implementations:

```rust
use std::ops::Mul;

#[derive(Debug, Clone, Copy)]
struct Vec3<T> {
    x: T,
    y: T,
    z: T,
}

// Vec3<T> * T — scale by any numeric type
impl<T: Mul<Output = T> + Copy> Mul<T> for Vec3<T> {
    type Output = Vec3<T>;
    fn mul(self, scalar: T) -> Vec3<T> {
        Vec3 {
            x: self.x * scalar,
            y: self.y * scalar,
            z: self.z * scalar,
        }
    }
}

fn main() {
    let v = Vec3 { x: 1.0_f64, y: 2.0, z: 3.0 };
    let scaled = v * 2.0;
    println!("{:?}", scaled);  // Vec3 { x: 2.0, y: 4.0, z: 6.0 }

    let v_int = Vec3 { x: 1, y: 2, z: 3 };
    let scaled_int = v_int * 10;
    println!("{:?}", scaled_int);  // Vec3 { x: 10, y: 20, z: 30 }
}
```

---

## Operator Overloading in Other Languages

Rust's approach to operator overloading — trait-based, explicit, no magic — sits in a specific spot on the design spectrum. Let's compare:

### C++: The Wild West

C++ allows overloading almost *any* operator, including some unusual ones:

```cpp
// C++ — extensive operator overloading
class Point {
public:
    double x, y;

    Point operator+(const Point& other) const {
        return {x + other.x, y + other.y};
    }

    // You can even overload -> , (), [], new, delete, <<, >>
    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        os << "(" << p.x << ", " << p.y << ")";
        return os;
    }
};
```

C++ lets you overload `new`, `delete`, `->`, `()`, type conversions, and more. This power has led to both brilliant libraries (Eigen for linear algebra) and impenetrable code.

### Python: Dunder Methods

Python uses "dunder" (double underscore) methods:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):       # +
        return Point(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):      # *
        return Point(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar):     # reverse * (scalar * point)
        return self.__mul__(scalar)

    def __getitem__(self, idx):     # []
        return (self.x, self.y)[idx]

    def __eq__(self, other):        # ==
        return self.x == other.x and self.y == other.y
```

Python's `__rmul__` is equivalent to Rust's `impl Mul<Point> for f64` — it handles the case where the left operand doesn't know how to multiply.

### Java: The Refusal

Java deliberately does **not** support operator overloading (except `+` for `String` concatenation, which is compiler magic):

```java
// Java — no operator overloading
BigDecimal a = new BigDecimal("1.5");
BigDecimal b = new BigDecimal("2.5");
BigDecimal c = a.add(b);  // Must use .add(), not +
// You CANNOT write: BigDecimal c = a + b;
```

**Why?** James Gosling (Java's creator) believed operator overloading leads to unreadable code. The downside: even simple math with `BigDecimal` or `Complex` numbers is painfully verbose.

### Kotlin: Operator Functions

Kotlin takes a similar approach to Rust — operators map to named functions:

```kotlin
data class Point(val x: Double, val y: Double) {
    operator fun plus(other: Point) = Point(x + other.x, y + other.y)
    operator fun times(scalar: Double) = Point(x * scalar, y * scalar)
    operator fun get(index: Int) = if (index == 0) x else y
}

val p = Point(1.0, 2.0) + Point(3.0, 4.0)  // Uses plus()
```

Very similar to Rust's trait-based approach, but Kotlin uses the `operator` keyword on regular methods rather than implementing a trait.

### Haskell: Type Classes

Haskell uses type classes, which are the direct ancestor of Rust's traits:

```haskell
data Point = Point Double Double

instance Num Point where
  (Point x1 y1) + (Point x2 y2) = Point (x1+x2) (y1+y2)
  (Point x1 y1) * (Point x2 y2) = Point (x1*x2) (y1*y2)
  -- Must define all Num methods...
```

Haskell goes even further — you can define **entirely new operators** like `<+>` or `>>=`. Rust doesn't allow this.

### Go: The Other Refusal

Go, like Java, does not support operator overloading at all:

```go
// Go — no operator overloading
type Point struct { X, Y float64 }

func (p Point) Add(other Point) Point {
    return Point{p.X + other.X, p.Y + other.Y}
}

// Must write: c := a.Add(b)
// Cannot write: c := a + b
```

Go's philosophy values simplicity and readability over expressiveness.

### Comparison Table

| Feature | Rust | C++ | Python | Java | Kotlin | Haskell | Go |
|---------|------|-----|--------|------|--------|---------|-----|
| Arithmetic operators | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| Comparison operators | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| Index `[]` | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Custom operators | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Overload `new`/alloc | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Overload `<<`/`>>` | ✅ (Shl/Shr) | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| Mechanism | Traits | Functions | Dunder methods | N/A | `operator` funs | Type classes | N/A |
| Compiler enforced | ✅ | Partial | ❌ | N/A | ✅ | ✅ | N/A |

> **Rust's sweet spot:** Enough operator overloading to write expressive mathematical code, but constrained enough (traits, no custom operators) to prevent the kind of abuse that gives C++ operator overloading a bad name.

---

## Real-World Examples

### Example 1: 2D Vector Math Library

A complete 2D vector type with full arithmetic support:

```rust
use std::ops::{Add, Sub, Mul, Neg, AddAssign, SubAssign};
use std::fmt;

#[derive(Clone, Copy, PartialEq)]
struct Vec2 {
    x: f64,
    y: f64,
}

impl Vec2 {
    fn new(x: f64, y: f64) -> Self {
        Vec2 { x, y }
    }

    fn magnitude(self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }

    fn normalized(self) -> Vec2 {
        let mag = self.magnitude();
        Vec2 { x: self.x / mag, y: self.y / mag }
    }

    fn dot(self, other: Vec2) -> f64 {
        self.x * other.x + self.y * other.y
    }
}

// Display
impl fmt::Display for Vec2 {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.2}, {:.2})", self.x, self.y)
    }
}

impl fmt::Debug for Vec2 {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Vec2({}, {})", self.x, self.y)
    }
}

// Vec2 + Vec2
impl Add for Vec2 {
    type Output = Vec2;
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

// Vec2 - Vec2
impl Sub for Vec2 {
    type Output = Vec2;
    fn sub(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x - rhs.x, y: self.y - rhs.y }
    }
}

// Vec2 * f64 (scale)
impl Mul<f64> for Vec2 {
    type Output = Vec2;
    fn mul(self, scalar: f64) -> Vec2 {
        Vec2 { x: self.x * scalar, y: self.y * scalar }
    }
}

// f64 * Vec2 (reverse scale)
impl Mul<Vec2> for f64 {
    type Output = Vec2;
    fn mul(self, vec: Vec2) -> Vec2 {
        Vec2 { x: self * vec.x, y: self * vec.y }
    }
}

// -Vec2
impl Neg for Vec2 {
    type Output = Vec2;
    fn neg(self) -> Vec2 {
        Vec2 { x: -self.x, y: -self.y }
    }
}

// Vec2 += Vec2
impl AddAssign for Vec2 {
    fn add_assign(&mut self, rhs: Vec2) {
        self.x += rhs.x;
        self.y += rhs.y;
    }
}

// Vec2 -= Vec2
impl SubAssign for Vec2 {
    fn sub_assign(&mut self, rhs: Vec2) {
        self.x -= rhs.x;
        self.y -= rhs.y;
    }
}

fn main() {
    let mut position = Vec2::new(0.0, 0.0);
    let velocity = Vec2::new(3.0, 4.0);
    let gravity = Vec2::new(0.0, -9.8);
    let dt = 0.016; // 60 FPS timestep

    // Physics update — reads like math!
    position += velocity * dt;
    position += gravity * (0.5 * dt * dt);

    println!("Position: {}", position);
    println!("Speed: {:.2}", velocity.magnitude());
    println!("Direction: {}", velocity.normalized());

    // Operator chaining
    let a = Vec2::new(1.0, 0.0);
    let b = Vec2::new(0.0, 1.0);
    let result = a + b * 2.0 + -a;  // (1,0) + (0,2) + (-1,0) = (0,2)
    println!("Result: {}", result);
}
```

### Example 2: Type-Safe Currency

A system where `USD + USD` works, but `USD + EUR` is a **compile-time error**:

```rust
use std::ops::Add;
use std::fmt;

#[derive(Debug, Clone, Copy)]
struct USD(f64);

#[derive(Debug, Clone, Copy)]
struct EUR(f64);

impl Add for USD {
    type Output = USD;
    fn add(self, other: USD) -> USD {
        USD(self.0 + other.0)
    }
}

impl Add for EUR {
    type Output = EUR;
    fn add(self, other: EUR) -> EUR {
        EUR(self.0 + other.0)
    }
}

impl fmt::Display for USD {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "${:.2}", self.0)
    }
}

impl fmt::Display for EUR {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "€{:.2}", self.0)
    }
}

fn main() {
    let price = USD(29.99);
    let tax = USD(2.40);
    let total = price + tax;
    println!("Total: {}", total);  // Total: $32.39

    let salary = EUR(3000.00);
    let bonus = EUR(500.00);
    let package = salary + bonus;
    println!("Package: {}", package);  // Package: €3500.00

    // This WON'T compile — type safety prevents currency mixing!
    // let oops = price + salary;
    // Error: expected `USD`, found `EUR`
}
```

This is the **newtype pattern** combined with operator overloading. The compiler prevents financial bugs that in other languages would only be caught at runtime (or not at all).

### Example 3: Custom Indexable Matrix

A matrix type with both single-index (row access) and tuple-index (element access):

```rust
use std::ops::{Index, IndexMut};
use std::fmt;

struct Matrix {
    data: Vec<f64>,
    rows: usize,
    cols: usize,
}

impl Matrix {
    fn new(rows: usize, cols: usize) -> Self {
        Matrix {
            data: vec![0.0; rows * cols],
            rows,
            cols,
        }
    }

    fn identity(size: usize) -> Self {
        let mut mat = Matrix::new(size, size);
        for i in 0..size {
            mat[(i, i)] = 1.0;
        }
        mat
    }
}

// Index by (row, col) tuple
impl Index<(usize, usize)> for Matrix {
    type Output = f64;
    fn index(&self, (row, col): (usize, usize)) -> &f64 {
        assert!(row < self.rows && col < self.cols,
                "Index ({}, {}) out of bounds for {}x{} matrix",
                row, col, self.rows, self.cols);
        &self.data[row * self.cols + col]
    }
}

impl IndexMut<(usize, usize)> for Matrix {
    fn index_mut(&mut self, (row, col): (usize, usize)) -> &mut f64 {
        assert!(row < self.rows && col < self.cols,
                "Index ({}, {}) out of bounds for {}x{} matrix",
                row, col, self.rows, self.cols);
        &mut self.data[row * self.cols + col]
    }
}

impl fmt::Display for Matrix {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for row in 0..self.rows {
            write!(f, "| ")?;
            for col in 0..self.cols {
                write!(f, "{:6.2} ", self[(row, col)])?;
            }
            writeln!(f, "|")?;
        }
        Ok(())
    }
}

fn main() {
    let mut mat = Matrix::new(3, 3);
    mat[(0, 0)] = 1.0;
    mat[(0, 1)] = 2.0;
    mat[(0, 2)] = 3.0;
    mat[(1, 0)] = 4.0;
    mat[(1, 1)] = 5.0;
    mat[(1, 2)] = 6.0;
    mat[(2, 0)] = 7.0;
    mat[(2, 1)] = 8.0;
    mat[(2, 2)] = 9.0;

    println!("Matrix:\n{}", mat);

    let identity = Matrix::identity(3);
    println!("Identity:\n{}", identity);
}
```

Output:
```
Matrix:
|   1.00   2.00   3.00 |
|   4.00   5.00   6.00 |
|   7.00   8.00   9.00 |

Identity:
|   1.00   0.00   0.00 |
|   0.00   1.00   0.00 |
|   0.00   0.00   1.00 |
```

---

## Guidelines — When to Overload

Operator overloading is powerful, but with power comes responsibility. Here's when to use it — and when not to.

### ✅ DO Overload When...

**1. The operator has natural mathematical meaning for your type:**
- Vectors: `+`, `-`, `*` (scalar), dot product
- Matrices: `+`, `-`, `*` (multiplication), `[]` (indexing)
- Complex numbers: `+`, `-`, `*`, `/`
- Fractions / Rationals: all arithmetic

**2. Your type wraps a numeric concept:**
- `Money(f64)` — addition and subtraction make sense
- `Temperature(f64)` — subtraction gives a difference, but adding two temperatures is debatable
- `Percentage(f64)` — multiplication with other numbers

**3. Indexing is natural for your container:**
- Custom collections: `HashMap`, `Matrix`, `Grid`
- Game boards: `board[(row, col)]`
- Databases: `table["column_name"]`

### ❌ DON'T Overload When...

**1. The operator has no intuitive meaning:**
```rust
// BAD — what does adding two Users even mean?
impl Add for User {
    type Output = User;
    fn add(self, other: User) -> User { ??? }
}
```

**2. The operation would surprise users:**
```rust
// BAD — multiplying two dates? What?
impl Mul for Date {
    type Output = Date;
    fn mul(self, other: Date) -> Date { ??? }
}
```

**3. You're overloading for a cute syntax hack:**
```rust
// BAD — using << as "append" like C++ streams
impl Shl<String> for Logger {
    type Output = Logger;
    fn shl(self, msg: String) -> Logger { /* log msg */ self }
}
// logger << "hello" << "world"  — clever, but confusing in Rust
```

### The Golden Rule

> **If someone reads `a + b` in your code, would they immediately understand what it does without checking the `Add` implementation?** If yes, overload. If no, use a named method instead.

| Decision | Example | Recommendation |
|----------|---------|----------------|
| `Vec2 + Vec2` | Vector addition | ✅ Overload `+` |
| `Matrix * Matrix` | Matrix multiplication | ✅ Overload `*` |
| `USD + USD` | Money addition | ✅ Overload `+` |
| `User + User` | ??? | ❌ Use a method |
| `Logger << msg` | Append to log | ❌ Use `.log(msg)` |
| `String + String` | Concatenation | ✅ (Already in std) |
| `Set + Set` | Union? | ⚠️ Maybe — `BitOr` for union is conventional |

---

## Exercises

### Exercise 1: Complex Number Arithmetic

Implement a `Complex` struct with `f64` real and imaginary parts. Implement:
- `Add` — `(a + bi) + (c + di) = (a+c) + (b+d)i`
- `Sub` — `(a + bi) - (c + di) = (a-c) + (b-d)i`
- `Mul` — `(a + bi) * (c + di) = (ac - bd) + (ad + bc)i`
- `Neg` — `-(a + bi) = (-a) + (-b)i`
- `Display` — print as `3 + 4i`, `3 - 4i`, or `3` (if imaginary is 0)

```rust
use std::ops::{Add, Sub, Mul, Neg};
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq)]
struct Complex {
    real: f64,
    imag: f64,
}

// YOUR CODE: implement Add, Sub, Mul, Neg, and Display

fn main() {
    let a = Complex { real: 3.0, imag: 4.0 };
    let b = Complex { real: 1.0, imag: -2.0 };

    println!("a = {}", a);        // 3 + 4i
    println!("b = {}", b);        // 1 - 2i
    println!("a + b = {}", a + b); // 4 + 2i
    println!("a - b = {}", a - b); // 2 + 6i
    println!("a * b = {}", a * b); // 11 - 2i
    println!("-a = {}", -a);       // -3 - 4i
}
```

<details>
<summary>💡 Solution</summary>

```rust
use std::ops::{Add, Sub, Mul, Neg};
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq)]
struct Complex {
    real: f64,
    imag: f64,
}

impl Add for Complex {
    type Output = Complex;
    fn add(self, rhs: Complex) -> Complex {
        Complex {
            real: self.real + rhs.real,
            imag: self.imag + rhs.imag,
        }
    }
}

impl Sub for Complex {
    type Output = Complex;
    fn sub(self, rhs: Complex) -> Complex {
        Complex {
            real: self.real - rhs.real,
            imag: self.imag - rhs.imag,
        }
    }
}

impl Mul for Complex {
    type Output = Complex;
    fn mul(self, rhs: Complex) -> Complex {
        Complex {
            real: self.real * rhs.real - self.imag * rhs.imag,
            imag: self.real * rhs.imag + self.imag * rhs.real,
        }
    }
}

impl Neg for Complex {
    type Output = Complex;
    fn neg(self) -> Complex {
        Complex { real: -self.real, imag: -self.imag }
    }
}

impl fmt::Display for Complex {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        if self.imag == 0.0 {
            write!(f, "{}", self.real)
        } else if self.imag > 0.0 {
            write!(f, "{} + {}i", self.real, self.imag)
        } else {
            write!(f, "{} - {}i", self.real, -self.imag)
        }
    }
}

fn main() {
    let a = Complex { real: 3.0, imag: 4.0 };
    let b = Complex { real: 1.0, imag: -2.0 };

    println!("a = {}", a);         // 3 + 4i
    println!("b = {}", b);         // 1 - 2i
    println!("a + b = {}", a + b); // 4 + 2i
    println!("a - b = {}", a - b); // 2 + 6i
    println!("a * b = {}", a * b); // 11 - 2i
    println!("-a = {}", -a);       // -3 - 4i
}
```

</details>

### Exercise 2: A Pixel Grid with Index

Create a `Grid<T>` struct that stores a 2D grid using a flat `Vec<T>`. Implement:
- `Index<(usize, usize)>` and `IndexMut<(usize, usize)>` for element access
- A `new(width, height, default: T)` constructor where `T: Clone`
- A `width()` and `height()` method

```rust
use std::ops::{Index, IndexMut};

struct Grid<T> {
    data: Vec<T>,
    width: usize,
    height: usize,
}

// YOUR CODE: implement new, width, height, Index, IndexMut

fn main() {
    let mut grid = Grid::new(10, 10, '.');

    grid[(0, 0)] = '#';
    grid[(9, 9)] = '#';
    grid[(4, 5)] = 'X';

    for row in 0..grid.height() {
        for col in 0..grid.width() {
            print!("{} ", grid[(row, col)]);
        }
        println!();
    }
}
```

<details>
<summary>💡 Solution</summary>

```rust
use std::ops::{Index, IndexMut};

struct Grid<T> {
    data: Vec<T>,
    width: usize,
    height: usize,
}

impl<T: Clone> Grid<T> {
    fn new(width: usize, height: usize, default: T) -> Self {
        Grid {
            data: vec![default; width * height],
            width,
            height,
        }
    }
}

impl<T> Grid<T> {
    fn width(&self) -> usize {
        self.width
    }

    fn height(&self) -> usize {
        self.height
    }
}

impl<T> Index<(usize, usize)> for Grid<T> {
    type Output = T;
    fn index(&self, (row, col): (usize, usize)) -> &T {
        assert!(row < self.height && col < self.width,
                "Grid index ({}, {}) out of bounds for {}x{} grid",
                row, col, self.width, self.height);
        &self.data[row * self.width + col]
    }
}

impl<T> IndexMut<(usize, usize)> for Grid<T> {
    fn index_mut(&mut self, (row, col): (usize, usize)) -> &mut T {
        assert!(row < self.height && col < self.width,
                "Grid index ({}, {}) out of bounds for {}x{} grid",
                row, col, self.width, self.height);
        &mut self.data[row * self.width + col]
    }
}

fn main() {
    let mut grid = Grid::new(10, 10, '.');

    grid[(0, 0)] = '#';
    grid[(9, 9)] = '#';
    grid[(4, 5)] = 'X';

    for row in 0..grid.height() {
        for col in 0..grid.width() {
            print!("{} ", grid[(row, col)]);
        }
        println!();
    }
}
```

</details>

### Exercise 3: Implement `Mul` for Matrix

Extend the `Matrix` type from the real-world examples to support matrix multiplication. Remember: for an $m \times n$ matrix multiplied by an $n \times p$ matrix, the result is an $m \times p$ matrix where:

$$C_{ij} = \sum_{k=0}^{n-1} A_{ik} \cdot B_{kj}$$

```rust
use std::ops::{Index, IndexMut, Mul};

// Use the Matrix struct from the real-world example above
// YOUR CODE: implement Mul<Matrix> for Matrix, with a panic if dimensions don't match

fn main() {
    let mut a = Matrix::new(2, 3);
    a[(0, 0)] = 1.0; a[(0, 1)] = 2.0; a[(0, 2)] = 3.0;
    a[(1, 0)] = 4.0; a[(1, 1)] = 5.0; a[(1, 2)] = 6.0;

    let mut b = Matrix::new(3, 2);
    b[(0, 0)] = 7.0;  b[(0, 1)] = 8.0;
    b[(1, 0)] = 9.0;  b[(1, 1)] = 10.0;
    b[(2, 0)] = 11.0; b[(2, 1)] = 12.0;

    let c = a * b;
    println!("Result:\n{}", c);
    // | 58.00  64.00 |
    // |139.00 154.00 |
}
```

<details>
<summary>💡 Solution</summary>

```rust
impl Mul for Matrix {
    type Output = Matrix;

    fn mul(self, rhs: Matrix) -> Matrix {
        assert_eq!(self.cols, rhs.rows,
                   "Cannot multiply {}x{} matrix by {}x{} matrix",
                   self.rows, self.cols, rhs.rows, rhs.cols);

        let mut result = Matrix::new(self.rows, rhs.cols);
        for i in 0..self.rows {
            for j in 0..rhs.cols {
                let mut sum = 0.0;
                for k in 0..self.cols {
                    sum += self[(i, k)] * rhs[(k, j)];
                }
                result[(i, j)] = sum;
            }
        }
        result
    }
}
```

</details>

---

## Summary

| Concept | Key Point |
|---------|-----------|
| **Operator = Trait** | Every operator in Rust maps to a trait in `std::ops` or `std::cmp` |
| **`Add`, `Sub`, `Mul`, `Div`, `Rem`** | Binary arithmetic — take `self` by value, return `Output` |
| **`Neg`, `Not`** | Unary operators — take `self` by value |
| **`AddAssign`, `SubAssign`, etc.** | Compound assignment — take `&mut self`, no `Output` |
| **`PartialEq`, `PartialOrd`** | Comparison — take `&self`, from `std::cmp` |
| **`Index`, `IndexMut`** | Subscript `[]` — return references |
| **Different types** | `impl Add<f64> for Point` — mix types freely |
| **Reverse direction** | `impl Mul<Point> for f64` — left operand is `self` |
| **Generic operators** | Use trait bounds: `impl<T: Add<Output=T>> Add for Vec3<T>` |
| **When to overload** | Only when the operator has natural, intuitive meaning |

### What You've Learned

- Operators in Rust are syntactic sugar for trait method calls
- How to implement `Add`, `Sub`, `Mul`, `Div`, `Rem`, and `Neg` for custom types
- The difference between `Add` (returns new value) and `AddAssign` (mutates in place)
- How to use `Index` and `IndexMut` to make types subscriptable
- How to overload operators between *different* types
- How Rust's operator overloading compares to C++, Python, Java, Kotlin, Haskell, and Go
- When operator overloading is appropriate — and when it isn't

---

**Previous:** [← Common Std Traits](./06-common-std-traits.md) · **Next:** [Supertraits →](./08-supertraits.md)

<p align="center">
  <i>Tutorial 7 of 9 — Stage 8: Generics & Traits</i>
</p>

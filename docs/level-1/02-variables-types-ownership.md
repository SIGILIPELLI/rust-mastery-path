# 02 · Variables, Types & Ownership Basics

Rust bindings are immutable by default — a design choice that pushes you
toward safer code, since the compiler catches accidental reassignment instead
of letting it silently happen. This module covers variables, the core types,
and the very first taste of Rust's signature feature: ownership.

## `let` bindings are immutable by default

```rust
fn main() {
    let age = 30;
    println!("Age: {}", age);   // Age: 30

    age = 31;   // ERROR: cannot assign twice to immutable variable `age`
}
```

That second assignment doesn't compile. `let` creates a binding you can read
but not change — this is the default, not an opt-in restriction.

## `let mut` for mutability

```rust
fn main() {
    let mut age = 30;
    println!("Age: {}", age);   // Age: 30

    age = 31;   // fine -- `mut` opts into reassignment
    println!("Age: {}", age);   // Age: 31
}
```

`mut` is a deliberate, visible marker in the source. Anyone reading `let mut`
knows this value is expected to change somewhere below.

## Shadowing

Shadowing lets you declare a *new* binding with the same name, optionally
with a different type — it's not mutation, it's a fresh variable that reuses
the name.

```rust
fn main() {
    let spaces = "   ";           // spaces: &str
    let spaces = spaces.len();    // spaces: usize -- a brand new binding
    println!("{}", spaces);       // 3

    let x = 5;
    let x = x + 1;   // new binding, shadows the old x
    let x = x * 2;   // new binding again
    println!("{}", x);   // 12
}
```

Shadowing is different from `mut`: each `let` creates a new variable (even if
it changes type), whereas `mut` reuses the same variable and requires the
same type throughout.

## Scalar types

Rust is statically typed, but the compiler infers types in most cases, so
annotations are usually optional.

```rust
fn main() {
    let a: i32 = -7;          // signed 32-bit integer (the default integer type)
    let b: u32 = 7;           // unsigned 32-bit integer -- no negative values
    let c: i64 = 9_000_000_000; // signed 64-bit, for bigger numbers
    let d: f64 = 3.14;        // 64-bit floating point (the default float type)
    let e: bool = true;       // boolean -- true or false
    let f: char = 'R';        // a single Unicode scalar value, in single quotes

    println!("{a} {b} {c} {d} {e} {f}");
    // -7 7 9000000000 3.14 true R
}
```

Integers come in `i8`/`u8` through `i128`/`u128`, plus `isize`/`usize` (sized
to the pointer width of the target machine). `i32` is the default when the
compiler can't infer otherwise, and `f64` is the default float. Note the
`{a}` shorthand inside `println!` — captured identifiers can be interpolated
directly into the format string.

## Compound types: tuples

A tuple groups values of *different* types into one fixed-size value.

```rust
fn main() {
    let person: (&str, i32, bool) = ("Alice", 30, true);

    // Destructuring
    let (name, age, active) = person;
    println!("{name} is {age}, active: {active}");
    // Alice is 30, active: true

    // Or access by index with dot notation
    println!("{}", person.0);   // Alice
    println!("{}", person.1);   // 30
}
```

## Compound types: arrays

An array is a fixed-length collection of values that all share the same type,
stored on the stack.

```rust
fn main() {
    let numbers: [i32; 4] = [10, 20, 30, 40];   // type: 4 i32s
    let zeros = [0; 5];                          // [0, 0, 0, 0, 0] -- repeat shorthand

    println!("{}", numbers[0]);        // 10
    println!("{}", numbers.len());     // 4
    println!("{:?}", zeros);           // [0, 0, 0, 0, 0] -- {:?} is the "debug" format
}
```

Unlike a `Vec` (covered in [Module 6](06-collections.md)), an array's length
is fixed at compile time and can't grow or shrink.

## Ownership fundamentals

This is Rust's core idea, and it's what lets the compiler guarantee memory
safety without a garbage collector. The rule at this level: **every value has
exactly one owner**, and when that owner goes out of scope, the value is
dropped (its memory freed).

For simple stack-only values (like integers), assigning to a new variable
just **copies** the bits — cheap, and both variables remain valid:

```rust
fn main() {
    let x = 5;
    let y = x;   // x is copied into y

    println!("x = {x}, y = {y}");   // x = 5, y = 5 -- both still usable
}
```

But for heap-allocated types like `String`, assignment **moves** ownership
instead of copying, because copying the underlying heap data on every
assignment would be expensive and Rust doesn't do it silently:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;   // ownership of the string data MOVES from s1 to s2

    println!("{s2}");   // hello -- fine, s2 owns it now

    // println!("{s1}");
    // ERROR: borrow of moved value: `s1`
    // value borrowed here after move
}
```

After the move, `s1` is no longer valid — the compiler statically forbids
using it, which prevents both variables from ever trying to free the same
memory. This is different from most languages, where `s2 = s1` would either
copy the string or leave two references to the same object with no
compile-time tracking of which is "still good."

## `.clone()` for an explicit deep copy

When you actually want two independent copies of heap data, call `.clone()`
— it's opt-in and visible, so a reader knows exactly where the (potentially
expensive) copy happens.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();   // explicit deep copy -- s1 stays valid

    println!("s1 = {s1}, s2 = {s2}");   // s1 = hello, s2 = hello
}
```

## `Copy` types vs non-`Copy` types

Types that are small, fixed-size, and live entirely on the stack (integers,
floats, `bool`, `char`, and tuples/arrays made only of `Copy` types)
implement the `Copy` trait — assigning them duplicates the value instead of
moving it, so the original stays usable. Heap-backed types like `String` and
`Vec<T>` are **not** `Copy`, because duplicating them isn't free — that's why
assignment moves instead.

```rust
fn main() {
    let a = 10;        // i32 is Copy
    let b = a;
    println!("{a} {b}");   // 10 10 -- fine, both valid

    let v1 = vec![1, 2, 3];   // Vec<i32> is NOT Copy
    let v2 = v1;
    // println!("{:?}", v1);  // ERROR: value borrowed after move
    println!("{:?}", v2);     // [1, 2, 3]
}
```

This is only the introduction to ownership — borrowing (`&` references),
lifetimes, and the full rule set get a dedicated deep dive in
[Level 2](../level-2/01-ownership-borrowing.md). For now, remember: each
value has one owner, assignment moves non-`Copy` values, and `.clone()` gets
you an explicit second copy when you need one.

## Cheat sheet

| Type | Category | Example | Copy? |
|------|----------|---------|-------|
| `i32`, `u32`, `i64`, ... | Signed/unsigned integer | `let x: i32 = -7;` | Yes |
| `f64`, `f32` | Floating point | `let pi: f64 = 3.14;` | Yes |
| `bool` | Boolean | `let ok = true;` | Yes |
| `char` | Single Unicode scalar | `let c = 'R';` | Yes |
| `(T1, T2, ...)` | Tuple | `let t = (1, "a", true);` | Yes, if all elements are |
| `[T; N]` | Fixed-size array | `let a = [1, 2, 3];` | Yes, if `T` is |
| `String` | Growable, heap-allocated text | `String::from("hi")` | No |
| `Vec<T>` | Growable, heap-allocated list | `vec![1, 2, 3]` | No |

## Exercise

Write a program that declares an immutable `let` binding for your name (a
`String`) and a `let mut` binding for a score (`i32`) starting at 0. Increase
the score twice using `+=`. Then create a tuple holding your name, your
score, and a `bool` for whether the score is above 50, destructure it into
three variables, and print them. Finally, create a `String`, move it into a
new variable, and add a comment showing the line that would fail to compile
if you tried to use the original.

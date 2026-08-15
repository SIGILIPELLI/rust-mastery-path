# 09 · Generics & Trait Objects

Rust gives you two different ways to write code that works over multiple
types: generics (`<T: Trait>`), resolved at compile time into a separate
copy per concrete type, and trait objects (`dyn Trait`), resolved at runtime
through a vtable. They look similar in a function signature but compile to
completely different machine code, with different tradeoffs. This module
builds the same "shape" abstraction both ways so the difference is visible.

## Generic functions: one function, many compiled copies

```rust
fn largest<T: PartialOrd + Copy>(items: &[T]) -> T {
    let mut max = items[0];
    for &item in items {
        if item > max {
            max = item;
        }
    }
    max
}

fn main() {
    let nums = vec![3, 7, 2, 9, 4];
    println!("largest int: {}", largest(&nums));

    let floats = vec![3.1, 7.7, 2.2];
    println!("largest float: {}", largest(&floats));
}
```

```text
largest int: 9
largest float: 7.7
```

`largest::<i32>` and `largest::<f64>` are two entirely separate functions in
the compiled binary — this is monomorphization. The trait bounds
(`PartialOrd + Copy`) are a compile-time contract: `T` must support `>` and
cheap bitwise copying, checked once at each call site, with zero cost at
runtime because there's no indirection at all — it's ordinary comparisons on
ordinary values.

## Static vs. dynamic dispatch on the same trait

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

struct Circle { radius: f64 }
impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
    fn name(&self) -> &str { "circle" }
}

struct Square { side: f64 }
impl Shape for Square {
    fn area(&self) -> f64 { self.side * self.side }
    fn name(&self) -> &str { "square" }
}

// Static dispatch: monomorphized, one copy of the function per concrete T.
fn describe_static<T: Shape>(shape: &T) -> String {
    format!("{} has area {:.2}", shape.name(), shape.area())
}

// Dynamic dispatch: one function, vtable lookup at runtime.
fn describe_dynamic(shape: &dyn Shape) -> String {
    format!("{} has area {:.2}", shape.name(), shape.area())
}

fn main() {
    let circle = Circle { radius: 2.0 };
    let square = Square { side: 3.0 };
    println!("{}", describe_static(&circle));
    println!("{}", describe_dynamic(&square));
}
```

```text
circle has area 12.57
square has area 9.00
```

`describe_static::<Circle>` and `describe_static::<Square>` are separate
compiled functions — same story as `largest`. `describe_dynamic` is a
*single* function that takes a fat pointer (data + vtable) and calls
`area()`/`name()` through the vtable at runtime; it works for any `Shape`
without being generic itself, at the cost of one indirect call per method.

## Where `dyn` earns its keep: heterogeneous collections

```rust
fn total_area(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> =
        vec![Box::new(Circle { radius: 1.0 }), Box::new(Square { side: 2.0 })];
    println!("total area: {:.2}", total_area(&shapes));
}
```

```text
total area: 7.14
```

`Vec<Box<dyn Shape>>` holds *different concrete types in the same
collection* — a `Circle` and a `Square` side by side. Generics can't express
this: `Vec<T>` is one `T` for the whole vector. This is the case where
`dyn Trait` isn't just "the slower option," it's the only option — you
genuinely don't know the concrete types until runtime, or there are too many
to monomorphize over.

## Trait bounds on impl blocks, not the struct

```rust
struct Wrapper<T> {
    value: T,
}

impl<T: std::fmt::Display> Wrapper<T> {
    fn show(&self) -> String {
        format!("Wrapper({})", self.value)
    }
}

fn main() {
    let w = Wrapper { value: 42 };
    println!("{}", w.show());
}
```

```text
Wrapper(42)
```

`Wrapper<T>` itself has no bound on `T` — you can build a `Wrapper` around
anything, `Display` or not. The bound lives on the `impl` block, so
`show()` only exists for `T: Display`. This is the idiomatic split: keep the
data structure maximally general, attach capability-specific methods only
where the capability is actually needed.

## Rust-specific traps

**Object safety silently blocks `dyn`.** A trait with a generic method, a
method returning `Self`, or a method taking `self` by value can't be used as
`dyn Trait` — the compiler says "the trait cannot be made into an object."
This usually surfaces long after a trait is designed, the first time someone
tries `Box<dyn YourTrait>` and the whole call site fails with an error that
doesn't point at which method is the problem.

**Monomorphization bloat.** Ten call sites of a generic function with ten
different concrete types produce ten copies of the compiled code — faster at
runtime, but it can measurably grow binary size and compile time on large
generic-heavy codebases. `dyn Trait` avoids this by having exactly one
compiled function; it's a real tradeoff, not just a style choice.

**`&dyn Trait` vs `Box<dyn Trait>` vs `Rc<dyn Trait>`.** All three are trait
objects, but they differ in who owns the pointee: `&dyn` borrows (needs a
place with a real lifetime), `Box<dyn>` owns on the heap, `Rc<dyn>`/`Arc<dyn>`
share ownership. Mixing them up produces borrow-checker errors that read
like lifetime problems but are really "you picked the wrong pointer kind for
how this value needs to be shared."

**Turbofish needed when inference can't pick `T`.** `largest(&nums)` above
infers `T` from `nums: Vec<i32>` — but `Vec::<i32>::new()` or a generic
function called with no argument that mentions `T` needs
`largest::<i32>(&[])` explicitly; the compiler can't always infer a type
parameter from context alone.

## Cheat sheet

| | Generics (`<T: Trait>`) | Trait objects (`dyn Trait`) |
|---|---|---|
| Dispatch | Static, resolved at compile time | Dynamic, vtable lookup at runtime |
| Binary size | One copy per concrete `T` | One shared implementation |
| Collections | `Vec<T>` — one concrete type | `Vec<Box<dyn Trait>>` — mixed types |
| Requires | Trait bound compiles per call site | Trait must be object-safe |
| Typical pointer | `&T`, owned `T` | `&dyn`, `Box<dyn>`, `Rc<dyn>`/`Arc<dyn>` |

## Exercise

Add a third shape, `Triangle { base: f64, height: f64 }`, implementing
`Shape`. Write a function `largest_shape(shapes: &[Box<dyn Shape>]) -> &dyn
Shape` that returns a reference to whichever shape has the greatest area,
using `dyn Shape` (not generics, since the input is already a mixed `Vec`).
Call it on a `Vec` containing all three shape types and print the winner's
`name()`.

# 03 · Traits & Generics

Traits and generics are how Rust writes code once and reuses it across many
types, without giving up compile-time checking or paying a runtime cost for
the flexibility. A **trait** describes behavior a type can implement (like an
interface); a **generic** lets a function or struct work over *any* type
that satisfies a given trait. Together they're the backbone of idiomatic
Rust — you've already used them indirectly every time you called `.clone()`
or `println!("{:?}", ...)`.

## Defining and implementing a trait

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article {
    headline: String,
    body: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}...", self.headline, &self.body[..20.min(self.body.len())])
    }
}

struct Tweet {
    username: String,
    text: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.username, self.text)
    }
}

fn main() {
    let article = Article {
        headline: String::from("Rust 2.0 announced"),
        body: String::from("The Rust team today announced a major release."),
    };
    let tweet = Tweet {
        username: String::from("ferris"),
        text: String::from("traits are awesome"),
    };

    println!("{}", article.summarize());
    println!("{}", tweet.summarize());
}
```

```text
Rust 2.0 announced: The Rust team today ...
@ferris: traits are awesome
```

Any type can implement any trait, as long as either the trait or the type is
defined in your own crate (the "orphan rule" — it prevents two unrelated
crates from both implementing the same foreign trait for the same foreign
type, which would create an ambiguity nobody could resolve).

## Default method implementations

```rust
trait Summary {
    fn title(&self) -> String;

    // A default implementation -- types can use this as-is or override it
    fn summarize(&self) -> String {
        format!("(Read more about {}...)", self.title())
    }
}

struct Article {
    headline: String,
}

impl Summary for Article {
    fn title(&self) -> String {
        self.headline.clone()
    }
    // no summarize() override -- uses the default
}

fn main() {
    let article = Article { headline: String::from("Rust 2.0 announced") };
    println!("{}", article.summarize());
}
```

```text
(Read more about Rust 2.0 announced...)
```

## Trait bounds: generic functions that require behavior

A generic function parameter like `<T>` means "works for any type" — but
"any type" alone doesn't let you call `.summarize()` inside the function,
because the compiler doesn't know `T` has that method. A **trait bound**
restricts `T` to types that implement a given trait, which is what unlocks
calling its methods:

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Tweet {
    text: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        self.text.clone()
    }
}

// `T: Summary` is the trait bound -- T can be any type that implements Summary
fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}

// Equivalent shorthand using `impl Trait` in argument position
fn notify_short(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

fn main() {
    let tweet = Tweet { text: String::from("Rust ships generics") };
    notify(&tweet);
    notify_short(&tweet);
}
```

```text
Breaking news! Rust ships generics
Breaking news! Rust ships generics
```

`fn notify<T: Summary>(item: &T)` and `fn notify_short(item: &impl Summary)`
compile to the same thing — `impl Trait` is sugar for a generic with a bound,
useful when you don't need to name `T` anywhere else in the signature.

## Multiple bounds and `where` clauses

```rust
use std::fmt::Debug;

trait Summary {
    fn summarize(&self) -> String;
}

// Multiple bounds with `+` -- T must implement both traits
fn notify_and_log<T: Summary + Debug>(item: &T) {
    println!("Breaking news! {}", item.summarize());
    println!("(debug: {:?})", item);
}

// The same signature, written as a `where` clause instead -- purely
// stylistic here, but it becomes much easier to read than `+`-chains
// once a function has several type parameters or several bounds each:
//
// fn notify_and_log<T>(item: &T) where T: Summary + Debug { ... }

#[derive(Debug)]
struct Tweet {
    text: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        self.text.clone()
    }
}

fn main() {
    let tweet = Tweet { text: String::from("where clauses read better") };
    notify_and_log(&tweet);
}
```

## Generic structs

```rust
struct Pair<T> {
    first: T,
    second: T,
}

impl<T: std::fmt::Display + PartialOrd> Pair<T> {
    fn new(first: T, second: T) -> Pair<T> {
        Pair { first, second }
    }

    fn larger(&self) -> &T {
        if self.first >= self.second { &self.first } else { &self.second }
    }
}

fn main() {
    let ints = Pair::new(5, 12);
    println!("larger: {}", ints.larger());   // larger: 12

    let floats = Pair::new(3.5, 1.2);
    println!("larger: {}", floats.larger());   // larger: 3.5

    let words = Pair::new(String::from("apple"), String::from("banana"));
    println!("larger: {}", words.larger());   // larger: banana
}
```

```text
larger: 12
larger: 3.5
larger: banana
```

`Pair<T>` is defined once but the compiler generates a specialized version
for each concrete `T` it's used with (`Pair<i32>`, `Pair<f64>`,
`Pair<String>`) — this is called **monomorphization**, and it's why generics
in Rust have no runtime cost: by the time the program runs, there's no
generic code left, only ordinary compiled functions per type.

## Trait objects: when you need a *collection* of different types

Generics pick one concrete type per call site — but sometimes you need a
single `Vec` holding several *different* types that share a trait. That's
what `dyn Trait` (a trait object) is for, at the cost of a small runtime
dispatch overhead instead of monomorphization:

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article { headline: String }
impl Summary for Article {
    fn summarize(&self) -> String { format!("Article: {}", self.headline) }
}

struct Tweet { text: String }
impl Summary for Tweet {
    fn summarize(&self) -> String { format!("Tweet: {}", self.text) }
}

fn main() {
    // Box<dyn Summary> -- a heap-allocated value of some type that impls Summary
    let items: Vec<Box<dyn Summary>> = vec![
        Box::new(Article { headline: String::from("Rust news") }),
        Box::new(Tweet { text: String::from("hello") }),
    ];

    for item in &items {
        println!("{}", item.summarize());
    }
}
```

```text
Article: Rust news
Tweet: hello
```

Rule of thumb: reach for generics (`<T: Trait>`) when you know the concrete
type at compile time and want zero-cost dispatch; reach for `dyn Trait` when
you need a heterogeneous collection or the type genuinely isn't known until
runtime (like items loaded from a plugin or config).

## Common standard-library traits worth knowing

| Trait | Purpose | Enables |
|-------|---------|---------|
| `Debug` | Programmer-facing formatting | `{:?}` in `println!` |
| `Display` | User-facing formatting | `{}` in `println!` |
| `Clone` | Explicit deep copy | `.clone()` |
| `PartialEq` / `Eq` | Equality comparison | `==`, `!=` |
| `PartialOrd` / `Ord` | Ordering comparison | `<`, `>`, `.sort()` |
| `Default` | A sensible zero-value | `T::default()`, `..Default::default()` |
| `Iterator` | Sequential production of values | `for` loops, adapters (next module) |

## Exercise

Define a trait `trait Shape { fn area(&self) -> f64; fn name(&self) -> &str; }`.
Implement it for a `struct Circle { radius: f64 }` and a
`struct Square { side: f64 }`. Write a generic function
`fn describe<T: Shape>(shape: &T)` that prints `"{name}: area = {area:.2}"`.
Then write `fn total_area(shapes: &[Box<dyn Shape>]) -> f64` that sums the
areas of a mixed `Vec<Box<dyn Shape>>` containing both circles and squares,
and print the total in `main`.

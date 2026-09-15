---
description: "Pattern Matching — Rust's match expression is exhaustive — the compiler forces you to handle every possible case, which eliminates an entire class of…"
---

# 08 · Pattern Matching

Rust's `match` expression is exhaustive — the compiler forces you to handle
every possible case, which eliminates an entire class of "forgot to check
this" bugs.

## Basic match

```rust
fn main() {
    let number = 3;

    match number {
        1 => println!("one"),
        2 => println!("two"),
        3 => println!("three"),
        _ => println!("something else"),   // `_` catches anything not listed
    }
}
```

```text
three
```

Removing the `_` arm here would be a compile error — `match` must cover every
possible value of the type being matched.

## Matching multiple values and ranges

```rust
fn main() {
    let number = 7;

    match number {
        1 | 2 | 3 => println!("small"),       // OR pattern
        4..=6 => println!("medium"),           // inclusive range
        7..=10 => println!("large"),
        _ => println!("out of range"),
    }
}
```

## Matching on `Option` and `Result`

```rust
fn describe(value: Option<i32>) -> String {
    match value {
        Some(n) if n < 0 => format!("negative: {}", n),   // match guard
        Some(0) => String::from("zero"),
        Some(n) => format!("positive: {}", n),
        None => String::from("nothing"),
    }
}

fn main() {
    println!("{}", describe(Some(-5))); // negative: -5
    println!("{}", describe(Some(0)));  // zero
    println!("{}", describe(Some(42))); // positive: 42
    println!("{}", describe(None));     // nothing
}
```

A `match guard` (`if n < 0` after the pattern) adds an extra condition —
the arm only matches if both the pattern *and* the guard are true.

## Destructuring structs and tuples

```rust
struct Point {
    x: i32,
    y: i32,
}

fn describe(p: &Point) -> &str {
    match p {
        Point { x: 0, y: 0 } => "origin",
        Point { x: 0, .. } => "on the y-axis",
        Point { y: 0, .. } => "on the x-axis",
        Point { .. } => "somewhere else",
    }
}

fn main() {
    println!("{}", describe(&Point { x: 0, y: 0 })); // origin
    println!("{}", describe(&Point { x: 0, y: 5 })); // on the y-axis
    println!("{}", describe(&Point { x: 3, y: 4 })); // somewhere else
}
```

`..` inside a pattern means "ignore the remaining fields" — useful when you
only care about matching a specific subset of a struct's fields.

## if let — for when you only care about one case

```rust
fn main() {
    let config_value: Option<i32> = Some(42);

    // Verbose with match:
    match config_value {
        Some(v) => println!("value is {}", v),
        None => {}   // nothing to do, but we still have to write this arm
    }

    // Cleaner with if let -- only handle the case you care about
    if let Some(v) = config_value {
        println!("value is {}", v);
    }
}
```

## while let — looping while a pattern keeps matching

```rust
fn main() {
    let mut stack = vec![1, 2, 3];

    while let Some(top) = stack.pop() {
        println!("{}", top);
    }
    // 3
    // 2
    // 1
}
```

`stack.pop()` returns `Option<T>` (`Some` while there are elements, `None`
once empty) — `while let` loops for as long as the pattern keeps matching
`Some`, stopping automatically at `None`.

## How It Actually Works

`match` compiles to a **decision tree**, not a linear chain of `if`
comparisons — the compiler analyzes all the arms together and builds an
efficient branching structure (often a jump table for simple enum
discriminants, or a tree of tests for structural patterns), so matching
against ten variants isn't ten sequential checks in the worst case. This is
also the same machinery that powers exhaustiveness checking: the compiler
literally constructs the space of possible values the scrutinee's type can
take and verifies your arms cover all of it, which is why adding a variant to
an enum turns every `match` on it elsewhere in the codebase that lacks a
wildcard `_` arm into a compile error instead of a silent runtime bug.

`if let Some(v) = config_value` and `while let` are sugar over that same
`match` machinery, just for the single-pattern case — `if let PAT = EXPR {
BODY }` desugars to `match EXPR { PAT => BODY, _ => {} }`. `while let Some(top)
= stack.pop()` desugars to a `loop` containing a `match` that `break`s on the
non-matching arm: `loop { match stack.pop() { Some(top) => { ... }, None =>
break } }`. Since `Vec::pop` returns `Option<T>` by *moving* the popped
element out rather than returning a reference, each iteration also
transfers ownership of `top` to the loop body — no separate bounds check or
null test is needed, because "empty" is represented as `None`, a value the
match has to handle explicitly rather than a sentinel that could be
forgotten.

## Cheat sheet

| Construct | When to use |
|-----------|-------------|
| `match` | Exhaustively handle every possible variant/value |
| `\|` (OR pattern) | Match several literal values in one arm |
| `a..=b` | Match an inclusive numeric range |
| Match guard (`if cond`) | Add an extra condition to a pattern |
| `if let` | Handle just one pattern, ignore the rest |
| `while let` | Loop for as long as a pattern keeps matching |
| `..` in a struct pattern | Ignore the remaining fields |

## 🔀 See this in another language

- [Ruby — Blocks, Procs & Lambdas Introduction](https://sigilipelli.github.io/ruby-mastery-path/level-1/08-blocks-procs-lambdas/)
- [R — Basic Plotting](https://sigilipelli.github.io/r-mastery-path/level-1/08-basic-plotting/)
- [Java — Exception Handling Basics](https://sigilipelli.github.io/java-mastery-path/level-1/08-exception-handling/)

## Exercise

Write a function `classify(n: i32) -> &'static str` that uses `match` with
ranges to return `"negative"`, `"zero"`, `"small"` (1-9), or `"large"` (10+).
Then write a loop using `while let` that repeatedly pops values off a
`Vec<Option<i32>>` and prints `"got: N"` for `Some(n)` values, skipping
`None` values silently, stopping when the vector is empty.

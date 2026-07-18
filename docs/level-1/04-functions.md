# 04 · Functions

Functions are declared with `fn`, and Rust's distinction between
**statements** and **expressions** shapes how you write them — understanding
that distinction makes the rest of the language click into place.

## Basic syntax

```rust
fn greet(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    greet("Alice");   // Hello, Alice!
    greet("Bob");     // Hello, Bob!
}
```

Parameters always require explicit type annotations (`name: &str`) — Rust
never infers a function's parameter or return types from how it's called,
unlike local variables.

## Return values

The idiomatic way to return a value is to make it the last **expression** in
the function body, with **no semicolon**:

```rust
fn square(x: i32) -> i32 {
    x * x   // no semicolon -- this expression's value is returned
}

fn main() {
    let result = square(6);
    println!("{result}");   // 36
}
```

`-> i32` declares the return type. You can also return early with the
explicit `return` keyword, typically for early exits:

```rust
fn absolute_value(x: i32) -> i32 {
    if x < 0 {
        return -x;   // early return, semicolon required here
    }
    x   // falls through if x >= 0, still no semicolon
}

fn main() {
    println!("{}", absolute_value(-7));   // 7
    println!("{}", absolute_value(4));    // 4
}
```

## Statements vs. expressions

This is one of Rust's most important structural ideas: a **statement**
performs an action and produces nothing (no value), while an **expression**
evaluates to a value. Adding a semicolon turns an expression into a
statement by discarding its value.

```rust
fn main() {
    let x = 5;          // `let x = 5;` is a statement -- it produces no value itself

    let y = {
        let a = 3;
        let b = 4;
        a * a + b * b   // no semicolon -- this is the block's value
    };

    println!("{y}");   // 25
}
```

That `{ ... }` block is itself an expression — it evaluates to whatever its
last unterminated line produces, and that's exactly the mechanism `if` as an
expression (see [Module 3](03-control-flow.md)) relies on. Add a semicolon
after `a * a + b * b` and the block would instead evaluate to `()`, the unit
value, and `y` would no longer be `25`.

## Nested / helper functions

Functions can be defined inside other functions, scoped to where they're
used. This is handy for small helpers that don't need to be visible anywhere
else.

```rust
fn main() {
    fn double(n: i32) -> i32 {
        n * 2
    }

    let values = [1, 2, 3, 4];
    for v in values.iter() {
        println!("{}", double(*v));
    }
    // 2
    // 4
    // 6
    // 8
}
```

`*v` dereferences the reference `.iter()` hands back to get the underlying
`i32` — you'll see `&`/`*` a lot more once borrowing is covered in depth in
[Level 2](../level-2/01-ownership-borrowing.md).

## Multiple parameters and returning early from logic

```rust
fn max_of_three(a: i32, b: i32, c: i32) -> i32 {
    let mut largest = a;
    if b > largest {
        largest = b;
    }
    if c > largest {
        largest = c;
    }
    largest   // expression, returned
}

fn main() {
    println!("{}", max_of_three(4, 9, 2));   // 9
}
```

## Functions that return nothing

A function with no `-> Type` implicitly returns `()`, the **unit type** — the
same value the discarded block above produced. It's Rust's equivalent of
"void," except it's an actual (zero-sized) type rather than a special case.

```rust
fn log_message(message: &str) {
    println!("[LOG] {message}");
    // implicitly returns () here
}

fn main() {
    log_message("starting up");   // [LOG] starting up
}
```

You may also encounter the `!` ("never") type on some functions — it marks a
function that never returns at all (for example, one that always panics or
loops forever). That's a niche detail you won't need to write yourself yet,
but it's why a function like `std::process::exit` can be used in places
expecting any type: it never returns, so it's compatible with everything.

## Cheat sheet

| Concept | Example | Notes |
|---------|---------|-------|
| Declaration | `fn add(a: i32, b: i32) -> i32 { a + b }` | Parameters and return type always explicit |
| Implicit return | `a + b` (no `;`) | Last expression in the body is the return value |
| Explicit return | `return a + b;` | Needed for early exits, requires a semicolon |
| No return value | `fn log(msg: &str) { ... }` | Implicitly returns `()`, the unit type |
| Block as expression | `let y = { ...; last_expr };` | A `{ }` block evaluates to its last unterminated expression |

## Exercise

Write a function `is_prime(n: u32) -> bool` that returns whether `n` is
prime (use a helper loop checking divisibility up to `n`). Write a second
function `classify(n: u32) -> &'static str` that returns `"prime"` or
`"composite"` by calling `is_prime`. In `main`, loop over the numbers 2
through 20 and print each number alongside its classification.

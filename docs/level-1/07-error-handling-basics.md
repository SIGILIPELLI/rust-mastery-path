# 07 · Error Handling Basics (Option, Result)

Rust has no `null` and no exceptions. Instead, the *type system* forces you to
deal with absence and failure explicitly, using two enums from the standard
library: `Option<T>` for "a value might not be there" and `Result<T, E>` for
"an operation might fail." The compiler won't let you forget to handle either
case — that's the whole point.

## `Option<T>` — a value that might not exist

```rust
fn main() {
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;

    println!("{}", some_number.is_some()); // true
    println!("{}", no_number.is_none());   // true

    // Finding something in a Vec returns an Option
    let names = vec!["Alice", "Bob", "Charlie"];
    let found = names.iter().find(|&&n| n == "Bob");
    println!("{:?}", found); // Some("Bob")

    let missing = names.iter().find(|&&n| n == "Zoe");
    println!("{:?}", missing); // None
}
```

```text
true
true
Some("Bob")
None
```

## `.unwrap()` and `.expect()` — and why to avoid them

```rust
fn main() {
    let some_number: Option<i32> = Some(5);
    println!("{}", some_number.unwrap()); // 5 -- fine, we know it's Some

    let no_number: Option<i32> = None;
    // no_number.unwrap();
    // ^ this would panic and crash the whole program:
    // thread 'main' panicked at 'called `Option::unwrap()` on a `None` value'

    // .expect() is the same, but with a custom panic message -- useful for
    // "this should never happen" invariants, still crashes the program.
    // no_number.expect("config value must be set");
}
```

`.unwrap()` and `.expect()` are fine in throwaway scripts, quick experiments,
or when you've already proven the value can't be `None`. In real code they're
a crash waiting to happen the moment your assumption is wrong — prefer the
safer patterns below whenever the caller could reasonably hit the empty case.

## Safe handling with `if let`

```rust
fn main() {
    let maybe_age: Option<i32> = Some(30);

    if let Some(age) = maybe_age {
        println!("Age is {age}");
    } else {
        println!("No age provided");
    }

    // unwrap_or / unwrap_or_else -- provide a fallback instead of panicking
    let missing_age: Option<i32> = None;
    let age = missing_age.unwrap_or(0);
    println!("age defaulted to {age}");

    let computed_default = missing_age.unwrap_or_else(|| {
        println!("computing a fallback...");
        18
    });
    println!("age defaulted to {computed_default}");
}
```

```text
Age is 30
age defaulted to 0
computing a fallback...
age defaulted to 18
```

Notice the `if let ... else` block only ran its `if` branch here since
`maybe_age` was `Some(30)` — the `else` branch (printing "No age provided")
would only fire for a `None` value, which is exactly what happens with
`missing_age` below.

## `Result<T, E>` — an operation that might fail

`Option` says "maybe nothing." `Result` says "maybe an error, and here's
*why*." Parsing text to a number is the classic example — it can fail if the
text isn't a valid number:

```rust
fn main() {
    let good: Result<i32, _> = "42".parse::<i32>();
    let bad: Result<i32, _> = "oops".parse::<i32>();

    match good {
        Ok(n) => println!("parsed: {n}"),
        Err(e) => println!("failed: {e}"),
    }

    match bad {
        Ok(n) => println!("parsed: {n}"),
        Err(e) => println!("failed: {e}"),
    }

    // unwrap_or works on Result too -- discards the error, keeps a default
    let value = "not a number".parse::<i32>().unwrap_or(-1);
    println!("value = {value}");
}
```

```text
parsed: 42
failed: invalid digit found in string
value = -1
```

## The `?` operator — a first taste

Inside a function that itself returns `Result`, the `?` operator unwraps an
`Ok` value or immediately returns the `Err` to the *caller* — no manual
`match` needed. This is the single most common error-handling pattern in
real Rust code, and you'll see much more of it in Level 2.

```rust
fn parse_and_double(input: &str) -> Result<i32, std::num::ParseIntError> {
    let n = input.parse::<i32>()?; // if this is Err, return it immediately
    Ok(n * 2)
}

fn main() {
    match parse_and_double("21") {
        Ok(result) => println!("doubled: {result}"),
        Err(e) => println!("error: {e}"),
    }

    match parse_and_double("abc") {
        Ok(result) => println!("doubled: {result}"),
        Err(e) => println!("error: {e}"),
    }
}
```

```text
doubled: 42
error: invalid digit found in string
```

`?` only works inside a function whose return type is `Result` (or `Option`)
with a compatible error type — the compiler enforces this, so if you forget,
you'll get a clear error telling you the function's return type doesn't
match.

## Cheat sheet

| Method | Works on | Behavior |
|--------|----------|----------|
| `.is_some()` / `.is_none()` | `Option` | Check without consuming |
| `.is_ok()` / `.is_err()` | `Result` | Check without consuming |
| `.unwrap()` | Both | Returns value, **panics** on `None`/`Err` |
| `.expect("msg")` | Both | Like `.unwrap()`, with a custom panic message |
| `.unwrap_or(default)` | Both | Returns value, or `default` on `None`/`Err` |
| `.unwrap_or_else(\|\| ...)` | Both | Returns value, or computes a fallback lazily |
| `if let Some(x) = opt` | `Option` | Handle just the "present" case |
| `if let Ok(x) = res` | `Result` | Handle just the "success" case |
| `?` | Both | Propagate `None`/`Err` out of the current function |

## Exercise

Write a function `fn safe_divide(a: f64, b: f64) -> Option<f64>` that returns
`None` if `b` is `0.0` and `Some(a / b)` otherwise. Call it with a few inputs
using `if let` to print either the result or a "cannot divide by zero"
message. Then write `fn parse_positive(input: &str) -> Result<i32, String>`
that parses `input` with `.parse::<i32>()`, returns `Err("not a
number".to_string())` on a parse failure, and `Err("must be positive"
.to_string())` if the parsed number is negative or zero, otherwise `Ok(n)`.

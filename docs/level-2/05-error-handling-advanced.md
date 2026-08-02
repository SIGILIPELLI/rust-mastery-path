# 05 · Error Handling Advanced

[Level 1](../level-1/07-error-handling-basics.md) covered `Option`, `Result`,
and a first taste of `?`. That's enough for small scripts, but a real
program usually has *several* things that can fail — file I/O, parsing,
network calls — each with its own error type. This module covers how to
design a custom error type that unifies them, implement the standard
`Error` trait so your errors play nicely with the rest of the ecosystem, and
use `?` across functions that return different error types.

## The problem: `?` requires matching error types

```rust
use std::num::ParseIntError;

fn double_from_str(input: &str) -> Result<i32, ParseIntError> {
    let n = input.parse::<i32>()?;
    Ok(n * 2)
}

fn main() {
    match double_from_str("21") {
        Ok(n) => println!("doubled: {n}"),
        Err(e) => println!("error: {e}"),
    }
}
```

That works for one error type. But the moment a function needs to use `?`
on *two different* fallible operations with *different* error types — say,
parsing a number and then reading a file — the compiler stops you, because
`?` converts the error type via `From`, and there's no automatic conversion
between two unrelated error types.

```rust
use std::num::ParseIntError;
use std::fs;
use std::io;

fn read_and_double(path: &str) -> Result<i32, ParseIntError> {
    let contents = fs::read_to_string(path)?;
    // ERROR: `?` couldn't convert `io::Error` into `ParseIntError`
    // the trait `From<io::Error>` is not implemented for `ParseIntError`
    let n = contents.trim().parse::<i32>()?;
    Ok(n * 2)
}
```

## Defining a custom error type

The fix is a single enum that can represent *any* error the function might
produce, with a `From` conversion for each source error type:

```rust
use std::fmt;
use std::fs;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(std::io::Error),
    Parse(ParseIntError),
}

// Display -- the user-facing error message
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "I/O error: {e}"),
            AppError::Parse(e) => write!(f, "parse error: {e}"),
        }
    }
}

// From impls -- these are what let `?` auto-convert into AppError
impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self {
        AppError::Io(e)
    }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self {
        AppError::Parse(e)
    }
}

fn read_and_double(path: &str) -> Result<i32, AppError> {
    let contents = fs::read_to_string(path)?;   // io::Error -> AppError via From
    let n = contents.trim().parse::<i32>()?;    // ParseIntError -> AppError via From
    Ok(n * 2)
}

fn main() {
    match read_and_double("does_not_exist.txt") {
        Ok(n) => println!("doubled: {n}"),
        Err(e) => println!("failed: {e}"),
    }
}
```

```text
failed: I/O error: No such file or directory (os error 2)
```

This is the core pattern of real-world Rust error handling: one enum per
"unit of fallible work" (often per module or per binary), a `From` impl per
source error, and `?` doing the conversion silently at every call site.

## Implementing `std::error::Error`

Implementing the standard `Error` trait (on top of `Debug` and `Display`,
both of which it requires) makes your error type interoperable with the rest
of the ecosystem — things like `Box<dyn Error>` and libraries that expect
"any standard error" will accept it.

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct ConfigError {
    message: String,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "config error: {}", self.message)
    }
}

impl Error for ConfigError {}   // Debug + Display are the only requirements

fn load_config(value: &str) -> Result<i32, ConfigError> {
    value.trim().parse::<i32>().map_err(|_| ConfigError {
        message: format!("'{value}' is not a valid number"),
    })
}

fn main() {
    match load_config("not a number") {
        Ok(n) => println!("config value: {n}"),
        Err(e) => println!("{e}"),
    }
}
```

```text
config error: 'not a number' is not a valid number
```

## `Box<dyn Error>` — the quick, flexible option

Writing a full custom enum for every function is sometimes overkill,
especially in `main` or small utilities. `Box<dyn Error>` lets a function
return "any error type at all," at the cost of losing the ability to
`match` on specific variants — you get a trait object instead of a concrete
enum:

```rust
use std::error::Error;
use std::fs;

fn read_and_double(path: &str) -> Result<i32, Box<dyn Error>> {
    let contents = fs::read_to_string(path)?;   // io::Error auto-boxes via `?`
    let n = contents.trim().parse::<i32>()?;    // ParseIntError auto-boxes too
    Ok(n * 2)
}

fn main() -> Result<(), Box<dyn Error>> {
    // `main` itself can return `Result` -- a returned `Err` prints via
    // `Debug` and exits with a non-zero status, which is handy for CLIs.
    match read_and_double("missing.txt") {
        Ok(n) => println!("doubled: {n}"),
        Err(e) => println!("failed: {e}"),
    }
    Ok(())
}
```

Rule of thumb: use `Box<dyn Error>` for application code (binaries, `main`,
one-off scripts) where callers just want to print and exit on failure; use a
custom enum for library code, where callers might want to `match` on
*which* error happened and react differently.

## `.map_err()` for one-off conversions

When you don't want a whole custom type, `.map_err()` transforms the error
variant of a `Result` inline — useful for adding context or converting to a
simpler type at a single call site:

```rust
fn parse_port(input: &str) -> Result<u16, String> {
    input
        .trim()
        .parse::<u16>()
        .map_err(|e| format!("invalid port '{input}': {e}"))
}

fn main() {
    match parse_port("70000") {
        Ok(port) => println!("port: {port}"),
        Err(e) => println!("{e}"),
    }
}
```

```text
invalid port '70000': number too large to fit in target type
```

## Panicking vs returning `Result`

Not every failure should be a `Result` — the two tools serve different
purposes:

| Situation | Use |
|-----------|-----|
| Caller can reasonably recover (bad input, missing file, network hiccup) | `Result<T, E>` |
| A bug/invariant violation that should never happen in correct code | `panic!` / `.expect("why")` |
| Library code, uncertain how the caller wants to react | `Result<T, E>` (let the caller decide) |
| Prototype/example code where clarity matters more than robustness | `.unwrap()` is acceptable |

`panic!` unwinds the stack and (by default) terminates the thread — it's for
"this program has a bug," not "this input was invalid." A library that
panics on bad user input instead of returning `Err` takes away the caller's
ability to handle it gracefully.

## Cheat sheet

| Tool | Use when |
|------|----------|
| `enum AppError { ... }` + `impl From<X> for AppError` | Multiple error sources, want to `match` on which one |
| `impl std::error::Error for E` | Making your error type interoperate with the ecosystem |
| `Box<dyn Error>` | Application code, don't need to distinguish error variants |
| `.map_err(\|e\| ...)` | One-off conversion or adding context at a single call site |
| `?` | Propagate any error convertible via `From` into the function's return type |
| `panic!` / `.expect()` | Unrecoverable bugs, not user-facing failures |

## Exercise

Define `enum CalcError { DivByZero, NegativeSqrt }` implementing `Debug`,
`Display`, and `std::error::Error`. Write
`fn safe_divide(a: f64, b: f64) -> Result<f64, CalcError>` and
`fn safe_sqrt(n: f64) -> Result<f64, CalcError>` (returning `NegativeSqrt`
for `n < 0.0`, otherwise `Ok(n.sqrt())`). Write a function
`fn compute(a: f64, b: f64) -> Result<f64, CalcError>` that divides `a` by
`b` with `?` and then takes the square root of the result with `?`, and call
it with a few inputs in `main`, printing either the result or the error.

# 06 · Testing in Rust

Rust has a testing framework built directly into the language and Cargo —
no external test runner to install, no separate config file to wire up.
Tests live alongside the code they exercise (or in a dedicated `tests/`
folder for higher-level checks), and `cargo test` finds and runs all of
them. This module covers writing unit tests, structuring integration tests,
and the assertion tools you'll use constantly from here on.

## Your first test

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;   // bring the outer module's items (like `add`) into scope

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 3), 5);
    }
}
```

```bash
cargo test
```

```text
running 1 test
test tests::adds_two_numbers ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

The pieces: `#[cfg(test)]` tells the compiler to only compile the `tests`
module when running tests (it's entirely absent from a normal `cargo build`
or `cargo run`), `mod tests` is just a regular module by convention, and
`#[test]` marks an individual function as a test case `cargo test` should
run and check for a panic.

## Assertion macros

```rust
fn is_even(n: i32) -> bool {
    n % 2 == 0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn checks_equality() {
        assert_eq!(2 + 2, 4);              // fails if not equal
        assert_ne!(2 + 2, 5);              // fails if equal
    }

    #[test]
    fn checks_boolean_conditions() {
        assert!(is_even(4));               // fails if false
        assert!(!is_even(3));              // fails if true
    }

    #[test]
    fn assert_with_custom_message() {
        let result = 2 + 2;
        // The message only prints if the assertion fails -- it's not
        // shown on success, so make it explain *what* went wrong.
        assert_eq!(result, 4, "expected 2 + 2 to equal 4, got {result}");
    }
}
```

`assert_eq!`/`assert_ne!` print both the left and right values on failure
(they require `Debug` on the compared type), which is why they're preferred
over a plain `assert!(a == b)` — a failing `assert!` only tells you the
condition was false, not what the actual values were.

## Testing that code panics

```rust
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("division by zero");
    }
    a / b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "division by zero")]
    fn divide_by_zero_panics() {
        divide(10, 0);
    }
}
```

`#[should_panic]` inverts the usual pass condition — the test passes *only
if* the function panics. The optional `expected = "..."` checks that the
panic message contains that substring, which guards against the test
accidentally passing because of a *different*, unrelated panic somewhere
else in the function.

## Testing `Result`-returning functions

Test functions can themselves return `Result<(), E>` instead of panicking on
failure — this lets you use `?` inside a test, which is often cleaner than
`.unwrap()` scattered everywhere:

```rust
use std::num::ParseIntError;

fn parse_and_double(input: &str) -> Result<i32, ParseIntError> {
    let n = input.parse::<i32>()?;
    Ok(n * 2)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn doubles_a_valid_number() -> Result<(), ParseIntError> {
        let result = parse_and_double("21")?;
        assert_eq!(result, 42);
        Ok(())
    }
}
```

A test returning `Err` counts as a failed test, printed with `Debug` — handy
for chaining several fallible setup steps with `?` instead of `.unwrap()`.

## Organizing tests: unit vs integration

| | Unit tests | Integration tests |
|---|---|---|
| Location | `#[cfg(test)] mod tests` inside the same file as the code | Separate files under `tests/` at the project root |
| Can see private items? | Yes (`use super::*;` has access) | No — only the crate's public API |
| Purpose | Verify one function/module in isolation | Verify the crate works as a whole, as an outside caller would use it |

```text
my_project/
    Cargo.toml
    src/
        lib.rs          -- contains #[cfg(test)] unit tests
    tests/
        integration.rs  -- a separate crate that only sees `pub` items
```

```rust
// tests/integration.rs
use my_project::add;   // only works if `add` is `pub` in src/lib.rs

#[test]
fn add_works_from_outside() {
    assert_eq!(add(2, 2), 4);
}
```

Each file directly under `tests/` is compiled as its own separate crate that
depends on your library — that's *why* it can only see `pub` items, and
also why splitting integration tests into multiple files is fine (they each
run independently, in parallel, by default).

## Running specific tests and controlling output

```bash
cargo test                    # run every test
cargo test adds_two_numbers   # run only tests whose name contains this string
cargo test -- --nocapture     # show println! output even for passing tests
cargo test -- --test-threads=1   # run tests sequentially (default is parallel)
```

Tests run in parallel by default and each gets its own thread — this is why
tests that touch shared state (like the same file on disk) can flake unless
you either give each test its own resource (e.g. a unique temp file name) or
force sequential execution.

## `#[ignore]` for slow tests

```rust
#[test]
#[ignore]   // skipped by default, e.g. a slow test hitting a real network call
fn expensive_integration_check() {
    // ...
}
```

```bash
cargo test              # skips ignored tests
cargo test -- --ignored # runs ONLY the ignored tests
```

## Cheat sheet

| Macro/attribute | Purpose |
|-----------------|---------|
| `#[test]` | Marks a function as a test case |
| `#[cfg(test)]` | Compiles a module only when testing |
| `assert!(cond)` | Fails if `cond` is false |
| `assert_eq!(a, b)` / `assert_ne!(a, b)` | Fails if not equal / equal, prints both values |
| `#[should_panic]` | Test passes only if the function panics |
| `#[ignore]` | Skips the test unless `--ignored` is passed |
| `cargo test <name>` | Run tests whose name contains `<name>` |
| `cargo test -- --nocapture` | Show `println!` output from passing tests |

## Exercise

Write a function `fn is_palindrome(s: &str) -> bool` (case-insensitive,
ignoring spaces) in a file. Add a `#[cfg(test)] mod tests` module with at
least four `#[test]` functions: one for a simple palindrome ("level"), one
for a phrase with spaces and mixed case ("A Man A Plan A Canal Panama"), one
for a non-palindrome, and one for an empty string (decide and assert what
the correct behavior should be). Run `cargo test -- --nocapture` and confirm
all four pass.

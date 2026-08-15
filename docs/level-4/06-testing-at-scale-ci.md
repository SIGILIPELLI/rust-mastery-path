# 06 · Testing at Scale & CI

A handful of `#[test]` functions is fine for one file. A real project needs
tests that scale — table-driven cases instead of one test per input,
`clippy` catching patterns that compile but shouldn't ship, `rustfmt`
keeping diffs reviewable, and all of it running automatically on every push
so nobody has to remember to run it locally. This module builds that
pipeline and shows real tool output, not just the commands.

## The code under test

```rust
pub fn parse_csv_row(row: &str) -> Vec<String> {
    row.split(',').map(|s| s.trim().to_string()).collect()
}

pub fn is_palindrome(s: &str) -> bool {
    let cleaned: String = s
        .chars()
        .filter(|c| c.is_alphanumeric())
        .map(|c| c.to_ascii_lowercase())
        .collect();
    cleaned.chars().eq(cleaned.chars().rev())
}
```

## Table-driven tests instead of one test per case

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_simple_row() {
        assert_eq!(parse_csv_row("a, b, c"), vec!["a", "b", "c"]);
    }

    #[test]
    fn palindrome_ignores_punctuation() {
        assert!(is_palindrome("A man, a plan, a canal: Panama"));
        assert!(!is_palindrome("not a palindrome"));
    }
}

#[cfg(test)]
mod proptests {
    use super::*;

    #[test]
    fn parse_csv_row_never_panics_on_edge_cases() {
        let cases = ["", ",", ",,,", "a,,b", "   spaced   ,x"];
        for case in cases {
            let result = parse_csv_row(case);
            assert!(!result.is_empty() || case.is_empty());
        }
    }
}
```

```text
$ cargo test
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`parse_csv_row_never_panics_on_edge_cases` loops over a table of edge cases
in a single `#[test]` — cheaper to write and maintain than five separate
functions, at the cost that a single failure doesn't tell you *which* case
failed without reading the assertion message. For a genuinely wide input
space, the `proptest` crate generates hundreds of random cases per run and
shrinks failures to a minimal reproducer automatically; the table-driven
version here is the lightweight version of the same idea with zero added
dependencies.

## `clippy`: catches things that compile but shouldn't

```bash
rustup component add clippy
cargo clippy
```

Here's real output against a function that clones a `Vec` just to call
`.len()` on the clone:

```rust
pub fn needless_clone_example(v: &Vec<i32>) -> usize {
    v.clone().len()
}
```

```text
warning: writing `&Vec` instead of `&[_]` involves a new object where a slice will do
  --> src/lib.rs:42:34
   |
42 | pub fn needless_clone_example(v: &Vec<i32>) -> usize {
   |                                  ^^^^^^^^^
   |
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.97.0/index.html#ptr_arg
   = note: `#[warn(clippy::ptr_arg)]` on by default
help: change this to
   |
42 ~ pub fn needless_clone_example(v: &[i32]) -> usize {
43 ~     v.to_owned().len()
   |
```

None of this is a compile error — `cargo build` accepts the function as-is.
`clippy` catches the *idiom* problem: taking `&Vec<T>` instead of `&[T]`
needlessly restricts callers (a caller with a plain array or slice now has
to build a `Vec` just to call this function), and cloning the whole vector
just to read its length is wasted work `v.len()` on the original reference
would have avoided entirely. This is exactly the class of bug that survives
code review by a tired human and gets caught instantly by a linter.

## `rustfmt`: consistent formatting, checkable in CI

```bash
rustup component add rustfmt
cargo fmt --check
```

```text
Diff in src/lib.rs:3:
 }
 
 pub fn is_palindrome(s: &str) -> bool {
-    let cleaned: String = s.chars().filter(|c| c.is_alphanumeric()).map(|c| c.to_ascii_lowercase()).collect();
+    let cleaned: String = s
+        .chars()
+        .filter(|c| c.is_alphanumeric())
+        .map(|c| c.to_ascii_lowercase())
+        .collect();
     cleaned.chars().eq(cleaned.chars().rev())
 }
```

`--check` exits nonzero and prints the diff without modifying the file —
the mode you want in CI, where the goal is "fail the build if formatting
drifted," not "silently reformat and hope someone commits the result."
Locally, plain `cargo fmt` applies the same diff in place.

## Wiring it into GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test
```

`cargo clippy -- -D warnings` is the detail that makes clippy actually
enforce anything in CI — without `-D warnings`, clippy's warnings print but
the command still exits `0`, and a red-warnings-but-green-CI build is worse
than no linting at all because it looks like it's being checked.

## Rust-specific traps

**`clippy` and `rustc` warnings are different sets.** A codebase can be
100% clean under `cargo build` (zero `rustc` warnings) and still have dozens
of `clippy::` lints — clippy is a separate, much larger rule set built on
top of the compiler's own diagnostics, not a superset the compiler already
runs.

**Doc tests run under `cargo test` too, and count as real test failures.**
Any ` ```rust ` fenced code block in a doc comment (`///`) is compiled and
run by `cargo test` by default. A doc comment with example code that
silently rotted (an API signature changed, the example still shows the old
one) fails CI the same as a broken `#[test]` — a common surprise the first
time it happens.

**`cargo fmt --check` and local `cargo fmt` can disagree if rustfmt
versions differ.** Formatting rules occasionally change between toolchain
versions; pinning a `rust-toolchain.toml` (or the CI action's `stable` vs. a
pinned version) keeps local and CI formatting decisions consistent, avoiding
a "works on my machine, fails in CI" formatting-only failure.

**Table-driven tests hide which case failed unless the assertion message
says so.** `assert!(!result.is_empty() || case.is_empty())` above reports
only that *some* iteration of the loop failed, not which `case` string
triggered it — worth adding `, "failed on case: {case:?}"` to `assert!` in
any loop-based test so the failure output is actually actionable.

## Cheat sheet

| Tool | Command | Catches |
|---|---|---|
| `cargo test` | Unit, integration, and doc tests | Logic bugs, regressions |
| `cargo clippy -- -D warnings` | Idiom / correctness lints | Non-idiomatic or subtly wasteful code |
| `cargo fmt --check` | Formatting drift | Inconsistent style, noisy diffs |
| `cargo test --doc` | Just doc-comment examples | Stale examples in `///` comments |
| `dtolnay/rust-toolchain@stable` | GitHub Actions setup step | Reproducible CI toolchain |

## Exercise

Add a fourth CI step, `cargo test --doc`, and write a doc comment on
`is_palindrome` with a ` ```rust ` example that calls it and asserts on the
result using `assert!` inside the doc block. Deliberately break the example
(assert the wrong boolean) and run `cargo test` locally to see the doc-test
failure output; then fix it and confirm all tests pass.

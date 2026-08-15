# 08 · Testing & Benchmarking

Rust builds unit testing into the language (`#[test]`, `cargo test`) and
leaves benchmarking to a crate (`criterion`) because measuring performance
correctly needs statistics `std` doesn't want to own. This module covers
both: writing tests that actually catch regressions, reading a real test
failure, and running a real criterion benchmark.

## The code under test

```rust
pub fn fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => {
            let (mut a, mut b) = (0u64, 1u64);
            for _ in 1..n {
                let next = a + b;
                a = b;
                b = next;
            }
            b
        }
    }
}

pub fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("division by zero".to_string())
    } else {
        Ok(a / b)
    }
}
```

## Unit tests live next to the code

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fibonacci_base_cases() {
        assert_eq!(fibonacci(0), 0);
        assert_eq!(fibonacci(1), 1);
    }

    #[test]
    fn fibonacci_tenth() {
        assert_eq!(fibonacci(10), 55);
    }

    #[test]
    fn divide_ok() {
        assert_eq!(divide(10, 2), Ok(5));
    }

    #[test]
    fn divide_by_zero_errs() {
        assert!(divide(1, 0).is_err());
    }

    #[test]
    #[should_panic(expected = "attempt to divide by zero")]
    fn raw_division_panics() {
        let b = std::hint::black_box(0);
        let _ = 1 / b;
    }
}
```

```text
$ cargo test
running 5 tests
.....
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`#[cfg(test)]` means this whole module is compiled only when testing — it
adds zero size or code to the real binary. `use super::*` pulls in the
parent module's items, which is why `fibonacci` and `divide` are visible
without a full path. `std::hint::black_box` in the panic test exists to stop
the compiler from constant-folding `1 / 0` at compile time and refusing to
build at all (see traps below).

## Reading a real failure

Here's what `fibonacci_tenth` looks like when the implementation is
deliberately broken (asserting the wrong expected value):

```text
running 6 tests
.... 4/6
tests::fibonacci_wrong_on_purpose --- FAILED
.

failures:

---- tests::fibonacci_wrong_on_purpose stdout ----

thread 'tests::fibonacci_wrong_on_purpose' (255833) panicked at src/lib.rs:60:9:
assertion `left == right` failed
  left: 5
 right: 999

failures:
    tests::fibonacci_wrong_on_purpose

test result: FAILED. 5 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
error: test failed, to rerun pass `--lib`
```

`left`/`right` in `assert_eq!` failures always mean "first argument" /
"second argument," not "expected"/"actual" — a common misread. Here `left:
5` is what `fibonacci(5)` actually returned; `right: 999` was the bogus
expectation, so the fix is in the test, not the function.

## Benchmarking with criterion

```bash
cargo add --dev criterion --features html_reports
```

```rust
// benches/fib_bench.rs
use criterion::{criterion_group, criterion_main, Criterion};
use std::hint::black_box;
use testing_demo::fibonacci;

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fibonacci 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);
```

```toml
# Cargo.toml
[[bench]]
name = "fib_bench"
harness = false
```

```text
$ cargo bench
Benchmarking fibonacci 20
Benchmarking fibonacci 20: Warming up for 3.0000 s
Benchmarking fibonacci 20: Collecting 100 samples in estimated 5.0000 s (713M iterations)
Benchmarking fibonacci 20: Analyzing
fibonacci 20            time:   [6.9764 ns 6.9809 ns 6.9855 ns]
                        change: [−0.6865% −0.4523% −0.2222%] (p = 0.00 < 0.05)
                        Change within noise threshold.
Found 12 outliers among 100 measurements (12.00%)
  6 (6.00%) low mild
  3 (3.00%) high mild
  3 (3.00%) high severe
```

`harness = false` tells cargo not to wrap this file with the default test
harness — criterion supplies its own `main` via `criterion_main!`. The
`[6.97ns 6.98ns 6.99ns]` triple is a 95% confidence interval, not a single
number; criterion runs statistical analysis across 100 samples specifically
so a one-off scheduling hiccup doesn't look like a regression. The "Change"
line only appears on a second run — criterion compares against the previous
run's saved baseline automatically.

## Rust-specific traps

**`1 / 0` as a literal is a compile error, not a panic.** Writing
`let _ = 1 / 0;` directly fails to build with `error: this operation will
panic at runtime` under `#[deny(unconditional_panic)]` — the compiler
const-evaluates literal division and refuses to ship code it knows will
crash. To actually exercise the runtime panic path in a test, the divisor
has to come from a non-const source; `std::hint::black_box(0)` is the
standard way to hide a value from constant folding.

**`black_box` moved from criterion to `std`.** Older criterion examples
import `criterion::black_box`; it now emits a deprecation warning telling
you to use `std::hint::black_box` instead, which is what test code (not
just benchmarks) should use too, since it's plain `std` with no dev-dependency
needed.

**Benchmarks don't run under `cargo test`.** `cargo test` executes anything
under `#[cfg(test)]` and files in `tests/`, but `benches/*.rs` only run
under `cargo bench`. It's easy to assume a passing `cargo test` validates
the whole project when performance-sensitive code has silently regressed.

**Integration tests (`tests/`) compile the crate as an external dependency.**
Anything not `pub` is invisible from `tests/*.rs` — unlike the `#[cfg(test)]`
module above, which lives inside the crate and can see private items via
`use super::*`. A function that works fine when unit-tested can fail to
compile once moved to an integration test, purely because it wasn't `pub`.

## Cheat sheet

| Command | Runs |
|---|---|
| `cargo test` | `#[test]` fns in `src/`, plus `tests/*.rs`, plus doc tests |
| `cargo test foo` | Only tests whose name contains `foo` |
| `cargo test -- --nocapture` | Also shows `println!` output from passing tests |
| `cargo bench` | `benches/*.rs`, using the harness declared in `Cargo.toml` |
| `#[should_panic(expected = "...")]` | Test passes only if the panic message contains the substring |
| `#[ignore]` | Skip by default; run explicitly with `cargo test -- --ignored` |

## Exercise

Add a `#[test]` for `divide` that checks `divide(i32::MIN, -1)` — integer
division overflow, since `i32::MIN / -1` doesn't fit in an `i32`. Run it
under `cargo test` and note whether it panics or returns an `Err`; then
explain in a comment above the test why `divide`'s current signature
(`Result<i32, String>`) does or doesn't already protect against this case.

# 05 · Security & Memory Safety

Rust eliminates whole classes of memory-safety bugs at compile time — use
after free, double free, data races on `&mut` data — just by being Rust.
That's not the whole security story: integer overflow, untrusted input, and
secrets lingering in memory are all still your responsibility, and Rust's
guarantees can even hide them if you don't know where to look. This module
covers three: checked arithmetic, input validation, and zeroing secrets.

## Setup

```bash
cargo add zeroize
```

## Integer overflow: `+` isn't as safe as it looks

```rust
fn safe_add_prices(prices: &[u32]) -> Option<u32> {
    prices.iter().try_fold(0u32, |acc, &p| acc.checked_add(p))
}

fn main() {
    let prices = vec![u32::MAX - 10, 20];
    match safe_add_prices(&prices) {
        Some(total) => println!("total: {total}"),
        None => println!("overflow detected, refusing to produce a wrong total"),
    }
}
```

```text
overflow detected, refusing to produce a wrong total
```

`checked_add` returns `None` on overflow instead of wrapping or panicking,
and `try_fold` stops the whole fold the first time any step returns `None`.
This is the shape you want for anything touching money, quantities, or
buffer sizes derived from untrusted input: a caller who summed two prices
near `u32::MAX` gets a clear "this overflowed," not a silently wrong total.

## Debug vs. release: the same `+` behaves differently

```rust
let a: u8 = 250;
let b: u8 = 10;
let result = std::panic::catch_unwind(|| a + b);
match result {
    Ok(v) => println!("sum: {v}"),
    Err(_) => println!("debug build: addition overflowed and panicked, as expected"),
}
```

Debug build:

```text
thread 'main' (327980) panicked at src/main.rs:48:46:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
debug build: addition overflowed and panicked, as expected
```

Release build (`cargo run --release`):

```text
sum: 4
```

Same source line, same inputs, two different outcomes. Debug builds insert
overflow checks by default and panic on overflow — a real safety net during
development. Release builds strip those checks for performance and fall back
to two's-complement wraparound: `250u8 + 10u8` becomes `4` silently. This is
one of the most consequential gaps between "it worked when I tested it" and
"it shipped a bug" in Rust — anything where overflow is a real possibility
(array indexing math, buffer sizes, money) needs explicit `checked_*`,
`wrapping_*`, or `saturating_*` calls, because you cannot rely on the debug
panic surviving into production.

## Validating untrusted input, not just parsing it

```rust
fn parse_percentage(input: &str) -> Result<u8, String> {
    let value: i64 = input.parse().map_err(|_| "not a number".to_string())?;
    if !(0..=100).contains(&value) {
        return Err(format!("{value} is out of range 0..=100"));
    }
    Ok(value as u8)
}

fn main() {
    for input in ["42", "150", "not-a-number", "-5"] {
        match parse_percentage(input) {
            Ok(p) => println!("{input} -> {p}%"),
            Err(e) => println!("{input} -> rejected: {e}"),
        }
    }
}
```

```text
42 -> 42%
150 -> rejected: 150 is out of range 0..=100
not-a-number -> rejected: not a number
-5 -> rejected: -5 is out of range 0..=100
```

Parsing into `i64` first (not `u8` directly) matters: parsing `"-5"`
straight into a `u8` fails with a generic "invalid digit" parse error that
doesn't distinguish "negative" from "garbage," losing information useful for
a caller. Parsing into a wider signed type, then explicitly range-checking,
gives you a real validation error message and avoids `u8::from_str`'s
implicit rejection of negatives looking the same as rejecting letters.

## Zeroing secrets instead of leaving them in memory

```rust
use zeroize::Zeroize;

struct ApiKey(String);

impl Drop for ApiKey {
    fn drop(&mut self) {
        self.0.zeroize();
    }
}

fn main() {
    let key = ApiKey("sk-super-secret-value".to_string());
    println!("using key of length {}", key.0.len());
} // ApiKey dropped here -- underlying String bytes are zeroed before deallocation
```

```text
using key of length 21
```

Normally, when a `String` is dropped, its heap buffer is freed but its bytes
aren't overwritten — the memory is returned to the allocator with the secret
still sitting in it until something else reuses that page. `zeroize()`
overwrites the buffer with zeros first, in a way the compiler is prevented
from optimizing away (ordinary "just set it to 0 then drop it" code can be
elided by the optimizer since it looks like dead writes). Wrapping this in a
custom `Drop` impl means every `ApiKey` gets zeroed automatically, at every
exit path — early return, panic unwind, normal scope end — without the
caller having to remember to call anything.

## Rust-specific traps

**Overflow checks are a debug-only safety net, not a production one.**
Reiterating because it's the single most common "it worked locally" bug
class: `cargo build --release` (and any real deployment) does not panic on
overflow by default. `overflow-checks = true` can be set explicitly in
`Cargo.toml`'s `[profile.release]` if you want the panic behavior in
production too, at a small performance cost.

**`unwrap()` on attacker-controlled input is a crash-as-a-service bug.**
`input.parse::<u8>().unwrap()` on a request body field turns "user sent
malformed data" into "the whole process panics," which — depending on your
error handling around it — can be a denial-of-service vector, not just an
ergonomics issue.

**`Drop` runs in reverse declaration order, which matters for zeroed
secrets referencing each other.** If one struct holds a reference derived
from another (rare with owned `String` secrets, but real with borrowed
buffers), the referent must not be zeroed before the last read of the
reference happens — Rust's drop order is deterministic and well-defined
(reverse of declaration), but it's still something to reason about
explicitly when secrets are nested.

**`Vec<u8>`/`String` reallocation can leave old copies behind.** A `String`
that grows past its capacity allocates a new, larger buffer and copies the
old bytes over — the *old* buffer, containing a stale copy of the secret,
is freed without being zeroed unless you're using a type designed to avoid
that (like `zeroize::Zeroizing<Vec<u8>>` or `secrecy::Secret<T>`), because
plain `String`/`Vec` growth isn't secret-aware.

## Cheat sheet

| Risk | Rust tool |
|---|---|
| Silent integer overflow in release | `checked_add`/`checked_sub`/`checked_mul`, or `saturating_*` |
| Untrusted input crashing the process | `Result`-returning parse + explicit range checks, never `unwrap()` |
| Secrets lingering in freed memory | `zeroize::Zeroize`, or a custom `Drop` calling it |
| Use-after-free, double-free | Prevented by ownership — not something to defend against manually |
| Data races on shared mutable state | Prevented by `Send`/`Sync` + the borrow checker |

## Exercise

Write a function `parse_transfer_amount(cents_str: &str, balance_cents:
u64) -> Result<u64, String>` that parses a string into `u64` cents,
rejects negative or non-numeric input, and additionally rejects any amount
greater than `balance_cents` using `checked_sub` (not a plain `-`) to
detect "would this overflow/underflow the resulting balance." Test it with
an amount larger than the balance and confirm you get a clean `Err`, not a
panic or a wrapped huge number.

# 02 · Performance Optimization

Rust's performance story is "zero-cost abstractions, if you use the right
one" — the language doesn't stop you from writing something slow, it just
gives you the tools to see exactly where the cost is coming from and a
faster alternative sitting right next to it. This module measures two real
tradeoffs — string building strategy and static vs. dynamic dispatch — with
`std::time::Instant`, in `--release` mode, because debug builds lie about
relative performance.

## Always benchmark in release mode

```bash
cargo run --release
```

Debug builds skip most optimization passes; a comparison that looks close
in `cargo run` can be 10-50x apart in `--release`, or reversed entirely.
Every number below is from `--release`.

## String building: allocate-per-iteration vs. reserve-once

```rust
// Allocates a new String per iteration via `+`.
fn build_report_allocating(items: &[&str]) -> String {
    let mut out = String::new();
    for item in items {
        out = out + item + ", ";
    }
    out
}

// Pre-reserves capacity, pushes instead of reallocating.
fn build_report_reserved(items: &[&str]) -> String {
    let approx_len: usize = items.iter().map(|s| s.len() + 2).sum();
    let mut out = String::with_capacity(approx_len);
    for item in items {
        out.push_str(item);
        out.push_str(", ");
    }
    out
}
```

```text
allocating concat: 449.458µs
reserved concat: 363.166µs
output length: 144000
```

`out + item + ", "` calls `String`'s `Add` impl, which consumes `out`,
allocates a new buffer sized for the combined length, copies both operands
in, and returns it — every single iteration, even though the final string
only needs one buffer. `String::with_capacity` allocates once upfront (an
estimate is fine; it grows if the estimate was low) and every `push_str`
after that is a plain memcpy into existing spare capacity. On 20,000 short
strings, that's the difference between ~20,000 allocations and effectively
one.

## Static vs. dynamic dispatch under load

```rust
// Vec<Box<dyn Fn>> - dynamic dispatch, heap allocation per closure.
fn sum_with_dyn(nums: &[i32], ops: &[Box<dyn Fn(i32) -> i32>]) -> i32 {
    nums.iter().map(|&n| ops.iter().fold(n, |acc, op| op(acc))).sum()
}

// Generic over a single closure type - static dispatch, no heap, inlinable.
fn sum_with_generic<F: Fn(i32) -> i32>(nums: &[i32], op: F) -> i32 {
    nums.iter().map(|&n| op(n)).sum()
}
```

```text
dyn dispatch sum: 3.165917ms
generic dispatch sum: 250.291µs
```

Over a million elements, the generic version is roughly 12x faster in this
run. `sum_with_generic` monomorphizes to a version of the function
specialized for the exact closure passed in — the compiler can see straight
through `op(n)` and inline it, sometimes vectorizing the whole loop.
`sum_with_dyn` calls through a vtable on every element; the compiler can't
see what's behind `Box<dyn Fn(i32) -> i32>` at the call site, so there's
nothing to inline, and each closure was also a separate heap allocation at
construction time. This is Level 3's static-vs-dynamic-dispatch module made
concrete with numbers: the *reason* to prefer generics isn't idiom, it's
that indirection has a real, measurable cost on hot paths.

## Rust-specific traps

**Debug-mode conclusions don't transfer.** In a debug build the dyn-vs-generic
gap above shrinks dramatically or can even invert, because neither version
gets inlining or vectorization — you're mostly measuring the interpreter-like
overhead debug builds have everywhere. Never tune based on `cargo run`
timings; always `--release`.

**`cargo bench` (criterion) vs. hand-rolled `Instant` timing.** The
hand-timed numbers above are single-run and noisy — fine for "which of these
two is obviously faster," not fine for "is this a 2% regression." Criterion
(Level 3, module 08) runs statistical sampling specifically because a single
`Instant::now()` measurement can vary run to run from OS scheduling, cache
state, and thermal throttling alone.

**Premature `Box<dyn Trait>` for "flexibility."** It's tempting to reach for
trait objects by default because they compile faster and avoid
monomorphization bloat — both real benefits — but on a genuinely hot loop
that difference in per-call cost adds up fast, as shown above. The right
default is generics when the concrete type is known at each call site, `dyn`
only when you actually need runtime polymorphism (heterogeneous collections,
plugin-style dispatch).

**`String` capacity estimates that undercount.** `approx_len` above assumes
every push exactly matches the reservation; if items vary a lot in length,
`with_capacity` still helps (fewer, larger reallocations) but doesn't
eliminate them entirely. Profiling, not guessing, is how you'd tune the
estimate for a real workload.

## Cheat sheet

| Technique | When it helps |
|---|---|
| `String::with_capacity(n)` | Building a string in a loop with a known/estimable final size |
| Generic `<F: Fn(...)>` over `Box<dyn Fn(...)>` | Hot loops calling the same closure repeatedly |
| `--release` builds | Any performance comparison, always |
| `cargo bench` / criterion | Precise, statistically sound measurement (Level 3, module 08) |
| `Vec::with_capacity(n)` | Same idea as `String::with_capacity`, for growable collections |
| Avoid `.clone()` in loops | Each clone is a real allocation + copy; borrow instead where possible |

## Exercise

Add a third string-building variant, `build_report_iter`, using
`items.join(", ")` (the standard library's own joiner). Time all three
versions on the same 20,000-item input and rank them. Then explain in a
comment why `join` is implemented the way it likely is internally — check
if its performance is closer to the allocating version or the
`with_capacity` version, and what that tells you about how `join` sizes its
buffer.

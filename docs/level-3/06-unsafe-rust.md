# 06 · Unsafe Rust

`unsafe` doesn't turn off the borrow checker or type checking — it unlocks
five specific operations the compiler can't otherwise verify: dereferencing
raw pointers, calling `unsafe fn`s, accessing mutable statics, implementing
`unsafe trait`s, and reading union fields. Everything else about Rust still
applies. The deal `unsafe` makes is: you prove the invariant by hand, the
compiler trusts you, and if you're wrong you get undefined behavior instead
of a compile error. This module writes real `unsafe` code and is explicit
about what each block is trusting.

## Splitting a slice into two mutable halves

The standard library's `slice::split_at_mut` can't be written in safe Rust —
the borrow checker sees one `&mut [T]` going in and refuses to believe two
non-overlapping `&mut` slices can come out, even though they provably don't
alias. This is the canonical example of `unsafe` encoding a fact the type
system is too conservative to see on its own.

```rust
use std::slice;

fn split_at_mut_manual(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();
    assert!(mid <= len);
    // SAFETY: `mid <= len`, so both resulting slices are within the
    // original allocation. The two slices [0, mid) and [mid, len) don't
    // overlap, so handing out two `&mut` into them simultaneously is sound
    // even though we bypass the borrow checker to construct them.
    unsafe {
        (
            slice::from_raw_parts_mut(ptr, mid),
            slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut nums = [1, 2, 3, 4, 5, 6];
    let (left, right) = split_at_mut_manual(&mut nums, 3);
    left[0] = 100;
    right[0] = 200;
    println!("{:?} {:?}", left, right);
    println!("{:?}", nums);
}
```

```text
[100, 2, 3] [200, 5, 6]
[100, 2, 3, 200, 5, 6]
```

The `assert!(mid <= len)` runs in *safe* code, before the `unsafe` block —
that's deliberate. The safety of `from_raw_parts_mut` depends on that bound
holding, so the check belongs outside the block where it's still enforced
unconditionally, not folded into a comment that trusts the caller.

## Mutable statics: `unsafe` on both read and write

```rust
static mut COUNTER: i32 = 0;

fn bump_counter() {
    // SAFETY: this program is single-threaded, so there is no data race on
    // COUNTER. Mutable statics are still `unsafe` to touch because the
    // compiler can't prove single-threaded access on its own.
    unsafe {
        COUNTER += 1;
        let val = std::ptr::addr_of!(COUNTER).read();
        println!("counter is now {val}");
    }
}

fn main() {
    bump_counter();
    bump_counter();
}
```

```text
counter is now 1
counter is now 2
```

## Rust-specific traps

**Shared references to `static mut` are now a hard compiler error.** Writing
`println!("{COUNTER}")` directly inside the `unsafe` block above — instead
of going through `std::ptr::addr_of!` — fails to compile on current Rust
with:

```text
error: creating a shared reference to mutable static
   = note: shared references to mutable statics are dangerous; it's
     undefined behavior if the static is mutated or if a mutable reference
     is created for it while the shared reference lives
```

This was legal (if dangerous) in older editions; the 2024 edition promotes
it to a hard error under `rust_2024_compatibility` lints because it was one
of the most common ways people wrote real UB while believing `unsafe`
"handled it." The fix is exactly what the example above does — read through
a raw pointer with `addr_of!`/`addr_of_mut!` instead of materializing a
reference to the static.

**`unsafe` blocks don't compose their guarantees.** Two `unsafe` blocks that
are each individually sound can be unsound *together* — e.g. one thread
calls `split_at_mut_manual` on overlapping index ranges from two different
call sites, each locally "obviously fine." `unsafe` invariants are a
property of the whole program's usage, not of a single block in isolation,
which is why every real `SAFETY:` comment should state the invariant the
*caller* must uphold, not just what the block itself does.

**Dangling raw pointers compile without complaint.** `let r = &x as *const
i32;` followed by `x` going out of scope leaves `r` a dangling pointer that
the compiler will let you store, pass around, and even print the address
of — the error only shows up if and when you `unsafe { *r }` it. Raw
pointers opt out of the lifetime tracking that makes references safe; nothing
checks a raw pointer's validity until the moment of dereference.

**`unsafe fn` doesn't mean "the whole body is exempt from safe-Rust
rules."** Marking a function `unsafe fn` only tells callers "read the docs
before calling this — there's a precondition." Inside the body, you still
need explicit `unsafe { }` blocks to actually perform pointer derefs or
other unsafe operations; the two `unsafe`s answer different questions
(caller's obligation vs. this specific operation's justification).

## Cheat sheet

| Operation | Why it needs `unsafe` |
|---|---|
| `*raw_ptr` | Compiler can't verify the pointer is valid and non-dangling |
| `unsafe fn foo()` call | Function has documented preconditions the compiler can't check |
| `static mut X` access | Compiler can't prove no data race across threads |
| `unsafe trait` impl | Implementer is asserting an invariant the trait promises |
| union field read | Compiler can't verify which variant is currently active |
| `std::ptr::addr_of!(X)` | Safe way to get a raw pointer to a static without a reference |

## Exercise

Write an `unsafe fn concat_first_bytes(a: &[u8], b: &[u8]) -> [u8; 2]` that
reads `a[0]` and `b[0]` via raw pointers (`a.as_ptr()`, `b.as_ptr()`) instead
of indexing, and document its precondition with a `/// # Safety` doc comment
stating both slices must be non-empty. Call it once correctly and print the
result; then write a second call site that would violate the precondition
(an empty slice) but comment it out, explaining in a code comment exactly
what undefined behavior would result if it ran.

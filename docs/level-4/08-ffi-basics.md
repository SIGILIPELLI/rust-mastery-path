# 08 · FFI Basics

Rust can call into C libraries and be called from them, using `extern "C"`
to opt into C's calling convention and ABI. Everything on the other side of
that boundary is invisible to the borrow checker — no lifetimes, no
ownership tracking, just raw memory and a promise. This module compiles a
tiny C function, calls it from Rust, exposes a Rust function back to C, and
walks a raw-pointer FFI call end to end.

## Setup: compiling C from `build.rs`

```bash
cargo add cc --build
```

```c
// cbits/mathlib.c
int add_ints(int a, int b) {
    return a + b;
}
```

```rust
// build.rs
fn main() {
    cc::Build::new().file("cbits/mathlib.c").compile("mathlib");
}
```

`build.rs` runs before your crate compiles; the `cc` crate wraps invoking
the system C compiler and produces a static library `cargo` then links into
your binary. This is how crates like `rusqlite --features bundled` (Level
3) compile a vendored C library instead of requiring it preinstalled.

## Calling C from Rust

```rust
use std::os::raw::c_int;

unsafe extern "C" {
    fn add_ints(a: c_int, b: c_int) -> c_int;
}

fn main() {
    // SAFETY: add_ints is a pure function of two ints, no shared state,
    // no pointers -- calling it has no preconditions beyond linking correctly.
    let sum = unsafe { add_ints(3, 4) };
    println!("3 + 4 via C = {sum}");
}
```

```text
3 + 4 via C = 7
```

`c_int` (not plain `i32`) is deliberate — it's a type alias that tracks
whatever the platform's C `int` actually is, which is `i32` on every common
target but isn't guaranteed to be by the C standard. Every `extern` function
call is `unsafe`, because the compiler has no way to verify the C side
actually behaves the way the signature claims — a mismatched signature
(wrong argument count, wrong types) is undefined behavior the type system
cannot catch.

## Exposing a Rust function to C

```rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_square(x: c_int) -> c_int {
    x * x
}
```

```text
5^2 via Rust-exported fn = 25
```

`#[unsafe(no_mangle)]` disables Rust's default name mangling, so the
compiled symbol is literally `rust_square` — required for C code (or any
other language's FFI) to find and call it by that name. `pub extern "C"`
does the reverse of the earlier `extern "C" { }` block: it says "call this
Rust function using the C calling convention," making it callable from
outside Rust.

## Raw pointers across the FFI boundary

```rust
let data = vec![1i32, 2, 3, 4, 5];
let total = unsafe { sum_via_ptr(data.as_ptr(), data.len()) };
println!("sum via raw pointer = {total}");

// SAFETY (documented on the function itself): caller must ensure `ptr` is
// valid for reads of `len` contiguous i32 values.
unsafe fn sum_via_ptr(ptr: *const i32, len: usize) -> i64 {
    let mut total: i64 = 0;
    for i in 0..len {
        total += unsafe { *ptr.add(i) } as i64;
    }
    total
}
```

```text
sum via raw pointer = 15
```

This is the shape a real C API for "give me a buffer and its length"
takes — `*const i32` plus a separate `usize` length, because C has no
concept of a Rust slice's fat pointer (pointer + length bundled together).
`data.as_ptr()` decays the `Vec<i32>` into a raw pointer, and `sum_via_ptr`
has to trust the caller that `len` accurately describes how many elements
are safely readable starting at `ptr` — nothing enforces that at the type
level once you're past `*const i32`.

## Rust-specific traps

**`extern` blocks and `#[no_mangle]` both require explicit `unsafe` as of
the 2024 edition.** Older Rust let you write `extern "C" { fn add_ints(...); }`
and `#[no_mangle]` without any `unsafe` keyword nearby. Current Rust
rejects both outright:

```text
error: extern blocks must be unsafe
error: unsafe attribute used without unsafe
  |
  = help: wrap the attribute in `unsafe(...)`
```

The fix is `unsafe extern "C" { ... }` and `#[unsafe(no_mangle)]` — the
2024 edition made explicit that *declaring* an FFI boundary is itself an
unsafe act (you're asserting the external signatures are correct), not just
*calling into* one.

**Raw pointer dereferences inside `unsafe fn` still need their own `unsafe`
block.** `sum_via_ptr` is an `unsafe fn`, but `*ptr.add(i)` inside its body
still needs `unsafe { }` around it under the current edition — the same
"unsafe fn doesn't make its whole body implicitly unsafe" rule from module
06, now showing up in a real FFI-shaped function.

**Layout mismatches between Rust structs and C structs.** A `struct` shared
across the FFI boundary needs `#[repr(C)]` — without it, Rust is free to
reorder fields for cache-friendliness, and a C caller reading the same bytes
with its own struct definition will read garbage. This bug produces no
compiler error on either side; it just silently reads the wrong field.

**String encoding isn't free at the boundary.** Rust `String`/`&str` are
guaranteed UTF-8 and not null-terminated; C strings are null-terminated and
encoding-agnostic. Passing a Rust string to C needs `CString`/`CStr`
conversion (`std::ffi::CString::new(s)`), not a raw pointer cast — a raw
cast compiles but hands C a buffer with no null terminator where it expects
one.

## Cheat sheet

| Direction | Mechanism |
|---|---|
| Call C from Rust | `unsafe extern "C" { fn foo(...); }`, call inside `unsafe { }` |
| Expose Rust to C | `#[unsafe(no_mangle)] pub extern "C" fn foo(...)` |
| Share struct layout | `#[repr(C)]` on the struct |
| Pass a string to C | `std::ffi::CString`, not a raw `&str` cast |
| Pass a buffer + length | `*const T` / `*mut T` plus a separate `usize` length |
| Compile a C source file | `cc` crate in `build.rs` |

## Exercise

Add a second C function, `int clamp(int value, int lo, int hi)`, to
`cbits/mathlib.c`, declare it in the `unsafe extern "C"` block, and call it
from `main` with a value outside the `[lo, hi]` range to confirm it clamps
correctly. Then write the `#[repr(C)]` struct `struct Point { x: c_int, y:
c_int }` and a C function `int point_sum(struct Point p)` that adds its
fields — verify the Rust side's field order must match the C struct
definition exactly, and explain what would happen if `#[repr(C)]` were
omitted.

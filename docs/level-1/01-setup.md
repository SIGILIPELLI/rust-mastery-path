# 01 · Setup & First Program

Rust programs are compiled ahead of time into native machine code by `rustc`,
the Rust compiler. In practice you rarely invoke `rustc` directly — you use
**Cargo**, Rust's build tool and package manager, which wraps compilation,
dependency management, and project scaffolding into one command-line tool.
Both are installed together via `rustup`, the official toolchain installer.

## Install rustup

```bash
# macOS / Linux
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows: download and run rustup-init.exe from https://rustup.rs
# (it installs the same toolchain plus the MSVC build tools it needs)
```

The installer downloads the compiler (`rustc`), the package manager (`cargo`),
and a few companion tools, then adds them to your shell's `PATH`. Restart your
terminal (or run the `source` command the installer prints) so the new `PATH`
takes effect.

Verify the install:

```bash
rustc --version
# rustc 1.8x.0 (xxxxxxxxx 2026-xx-xx)

cargo --version
# cargo 1.8x.0 (xxxxxxxxx 2026-xx-xx)
```

`rustc` compiles a single source file into an executable; `cargo` manages
entire **projects** — compiling, running, testing, formatting, and pulling in
third-party libraries (called *crates*). From here on, we'll use Cargo almost
exclusively.

## Creating a project with Cargo

```bash
cargo new hello_rust
```

```
     Created binary (application) `hello_rust` package
```

This creates a new directory called `hello_rust` with everything a minimal
project needs already in place:

```
hello_rust/
├── Cargo.toml
└── src/
    └── main.rs
```

`Cargo.toml` is the project's manifest — metadata and dependencies, written in
TOML:

```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
```

`src/main.rs` is the entry-point source file, and Cargo has already filled it
in with a working program:

```rust
fn main() {
    println!("Hello, world!");
}
```

`cargo new` also initializes a git repository and a `.gitignore` (which
excludes the `target/` build output directory) unless you're already inside
one.

## Running the program

From inside the project directory:

```bash
cd hello_rust
cargo run
```

```
   Compiling hello_rust v0.1.0 (/path/to/hello_rust)
    Finished dev [unoptimized + debuginfo] target(s) in 0.42s
     Running `target/debug/hello_rust`
Hello, world!
```

`cargo run` compiles the project (if anything changed since the last build)
and then immediately runs the resulting binary. It's the command you'll reach
for constantly during development.

## `cargo build` vs `cargo run` vs `cargo build --release`

```bash
cargo build
# Compiles only -- produces target/debug/hello_rust, does NOT run it.

cargo build --release
# Compiles with optimizations turned on -- produces target/release/hello_rust.
# Release builds are slower to compile but much faster to run; use them for
# anything you'd actually ship or benchmark.
```

| Command | Compiles? | Runs? | Output location | Optimized? |
|---------|-----------|-------|------------------|------------|
| `cargo build` | Yes (if needed) | No | `target/debug/` | No |
| `cargo run` | Yes (if needed) | Yes | `target/debug/` | No |
| `cargo build --release` | Yes (if needed) | No | `target/release/` | Yes |

During day-to-day learning and small projects, `cargo run` is all you need.
Reach for `--release` once you care about runtime speed.

## Anatomy of the program

| Piece | Meaning |
|-------|---------|
| `fn main()` | The program's entry point — Cargo always looks for a `main` function in `src/main.rs`. |
| `println!("Hello, world!")` | Prints text followed by a newline. The trailing `!` marks this as a **macro** call, not a plain function call. |
| `;` | Every statement ends with a semicolon. |
| `{ }` | Curly braces delimit blocks — function bodies, loop bodies, and more. |

The `!` after `println` isn't decoration — `println!` is a macro that expands
at compile time to generate the actual formatting and printing code. You'll
meet other macros like `vec!` and `format!` soon; anything ending in `!` is a
macro invocation, not a regular function.

## Choosing an editor

**VS Code with the `rust-analyzer` extension** is the standard choice for
Rust development — it's free, and `rust-analyzer` gives you inline type
hints, autocomplete, jump-to-definition, and real-time error checking that
matches what the compiler itself would say. Install VS Code, then install
"rust-analyzer" from the Extensions panel, and it will auto-detect any Cargo
project you open. (RustRover from JetBrains is a solid paid alternative if
you prefer a full IDE.)

## Exercise

Use `cargo new greeter` to scaffold a new project. Edit `src/main.rs` so
`main` prints a greeting for three different names, one `println!` call per
name. Run it with `cargo run`, then run `cargo build --release` and confirm
the optimized binary exists at `target/release/greeter`.

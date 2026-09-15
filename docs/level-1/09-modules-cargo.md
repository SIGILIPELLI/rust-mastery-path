---
description: "Modules & Cargo Project Structure — As a project grows past one file, you need a way to organize code into named groups and control what's visible from…"
---

# 09 · Modules & Cargo Project Structure

As a project grows past one file, you need a way to organize code into named
groups and control what's visible from where. Rust does this with modules,
and Cargo manages the project/build structure around them.

## The anatomy of a Cargo project

```bash
cargo new my_project
cd my_project
```

```text
my_project/
    Cargo.toml       -- project metadata and dependencies
    src/
        main.rs      -- entry point for a binary crate
```

```toml
# Cargo.toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"

[dependencies]
# crates (packages) go here, e.g.:
# serde = "1.0"
```

```bash
cargo build     # compiles the project
cargo run       # compiles and runs it
cargo test      # runs tests
```

## Defining a module inline

```rust
mod math {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }

    fn internal_helper() -> i32 {
        42   // not `pub` -- invisible outside this module
    }

    pub fn double(a: i32) -> i32 {
        add(a, a)   // modules can freely use their own private items
    }
}

fn main() {
    println!("{}", math::add(2, 3));    // 5
    println!("{}", math::double(4));    // 8
    // math::internal_helper();  -- ERROR: private, not accessible here
}
```

Everything in a module is private by default — you must mark items `pub` to
expose them outside the module. This is the opposite default from most
languages, and it's deliberate: it forces you to think about what's actually
part of your public API.

## Splitting modules into separate files

```text
src/
    main.rs
    math.rs
```

```rust
// src/math.rs
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

pub fn subtract(a: i32, b: i32) -> i32 {
    a - b
}
```

```rust
// src/main.rs
mod math;   // tells Rust to look for src/math.rs (or src/math/mod.rs)

fn main() {
    println!("{}", math::add(2, 3));      // 5
    println!("{}", math::subtract(5, 2)); // 3
}
```

## Nested modules and paths

```rust
mod shapes {
    pub mod circle {
        pub fn area(radius: f64) -> f64 {
            std::f64::consts::PI * radius * radius
        }
    }

    pub mod square {
        pub fn area(side: f64) -> f64 {
            side * side
        }
    }
}

fn main() {
    println!("{:.2}", shapes::circle::area(2.0));  // 12.57
    println!("{:.2}", shapes::square::area(3.0));   // 9.00
}
```

## `use` — bringing paths into scope

```rust
mod shapes {
    pub mod circle {
        pub fn area(radius: f64) -> f64 {
            std::f64::consts::PI * radius * radius
        }
    }
}

use shapes::circle;   // now `circle::area(...)` works without the full path

fn main() {
    println!("{:.2}", circle::area(2.0)); // 12.57
}
```

## Adding a dependency (crate) to Cargo.toml

```toml
[dependencies]
rand = "0.8"
```

```rust
use rand::Rng;

fn main() {
    let mut rng = rand::thread_rng();
    let n: u32 = rng.gen_range(1..=100);
    println!("Random number: {}", n);
}
```

Running `cargo build` (or `cargo run`) after adding a line to `[dependencies]`
automatically downloads and compiles the crate from
[crates.io](https://crates.io) — no separate install step needed.

## How It Actually Works

Modules (`mod`) are a purely compile-time namespacing and visibility
mechanism — they don't correspond to separate compiled units, dynamic
libraries, or runtime lookups the way packages do in some languages. The
compiler resolves every `use` path and privacy check (`pub` vs private)
during compilation and then erases the module structure entirely; at the
machine-code level there's no notion of "module boundary" left, just
functions and data laid out by the optimizer. This is why moving code
between modules never affects performance — it's a source-organization
concept that disappears before codegen.

`cargo build` resolving `rand = "0.8"` involves two separate files working
together: `Cargo.toml` states version *requirements* (semver ranges), while
`Cargo.lock` records the exact versions actually resolved and downloaded —
committing `Cargo.lock` for a binary project is what makes builds
reproducible across machines and time, since re-resolving `"0.8"` a year
later could otherwise pick a different patch release. Each dependency crate
is compiled from source on your machine (crates.io distributes source, not
prebuilt binaries) and then, critically, **statically linked** into your
final binary by default — there's no `rand.dll`/`.so` your program loads at
runtime, no dependency-resolution step at program startup. That's a direct
consequence of Rust's ahead-of-time, whole-program-optimizing compilation
model: the compiler can inline and optimize across crate boundaries because
everything is available as source (or pre-compiled `.rlib` artifacts) at
build time.

## Cheat sheet

| Task | Command/Syntax |
|------|-----------------|
| New project | `cargo new project_name` |
| Build | `cargo build` |
| Build and run | `cargo run` |
| Run tests | `cargo test` |
| Define a module | `mod name { ... }` or a separate `name.rs` file |
| Expose an item | `pub fn ...` / `pub struct ...` |
| Bring a path into scope | `use path::to::item;` |
| Add a dependency | Edit `[dependencies]` in `Cargo.toml` |

## 🔀 See this in another language

- [Ruby — Gems & Bundler Basics](https://sigilipelli.github.io/ruby-mastery-path/level-1/09-gems-bundler/)
- [R — Packages](https://sigilipelli.github.io/r-mastery-path/level-1/09-packages/)
- [Java — Packages & Build Tools Intro](https://sigilipelli.github.io/java-mastery-path/level-1/09-packages-build-tools/)

## Exercise

Create a new Cargo project. Add a module `inventory` (as a separate
`src/inventory.rs` file) with a `pub struct Item { pub name: String, pub
quantity: u32 }` and a `pub fn total_value(items: &[Item], price_per_unit:
f64) -> f64`. In `main.rs`, `use` the module, build a `Vec<Item>` with a few
entries, and print the total value.

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

## Exercise

Create a new Cargo project. Add a module `inventory` (as a separate
`src/inventory.rs` file) with a `pub struct Item { pub name: String, pub
quantity: u32 }` and a `pub fn total_value(items: &[Item], price_per_unit:
f64) -> f64`. In `main.rs`, `use` the module, build a `Vec<Item>` with a few
entries, and print the total value.

# 09 · Crates & Dependency Management

A **crate** is Rust's unit of compilation and distribution — your own
project is a crate, and every reusable library you pull in from
[crates.io](https://crates.io) is a crate too. [Level 1](../level-1/09-modules-cargo.md)
showed how to add one dependency line and use it; this module goes deeper
into how Cargo resolves versions, what semantic versioning actually
guarantees, how `Cargo.lock` fits in, and how workspaces organize a project
that's grown into multiple crates.

## Anatomy of a dependency line

```toml
[dependencies]
rand = "0.8"
serde = { version = "1.0", features = ["derive"] }
```

`"0.8"` isn't an exact pin — it's shorthand for a semantic versioning
**range**. Understanding that range is the single most useful piece of
Cargo knowledge for avoiding "why did my build suddenly break" surprises.

## Semantic versioning: what `"0.8"` actually means

A version is `MAJOR.MINOR.PATCH`. The promise behind semver is: patch
releases only fix bugs, minor releases only add backward-compatible
features, and major releases may break compatibility. Cargo's default
caret requirement (`"0.8"` is shorthand for `"^0.8"`) uses that promise to
pick the *widest safe range*:

| Requirement | Matches | Excludes |
|-------------|---------|----------|
| `"1.2.3"` (same as `"^1.2.3"`) | `>=1.2.3, <2.0.0` | `2.0.0` and later |
| `"0.8"` (same as `"^0.8.0"`) | `>=0.8.0, <0.9.0` | `0.9.0` and later |
| `"=1.2.3"` | Exactly `1.2.3` | Everything else |
| `"~1.2"` | `>=1.2.0, <1.3.0` | Even patch-only updates beyond 1.2.x conceptually align with `^1.2`, but `~` is stricter about the minor version |

Note the special case: for `0.x` versions, semver treats the **minor**
number like a major version bump (pre-1.0 crates are considered unstable by
convention), which is why `^0.8` only allows `0.8.x`, not `0.9.0`.

```bash
cargo add rand@0.8       # adds rand = "0.8" to [dependencies]
cargo update             # updates dependencies within their allowed ranges
cargo update -p rand     # updates only the `rand` crate
cargo outdated           # (needs `cargo install cargo-outdated`) shows what's behind
```

## `Cargo.lock`: reproducible builds

`Cargo.toml` states a *range* of acceptable versions; `Cargo.lock` records
the *exact* versions Cargo actually resolved and used, including transitive
dependencies (dependencies of your dependencies). This is what makes builds
reproducible: everyone who builds the project with the same `Cargo.lock`
gets bit-for-bit the same dependency tree, regardless of what's been
published to crates.io since.

| Project type | Commit `Cargo.lock`? |
|---------------|------------------------|
| Binary (an application, has `src/main.rs`) | Yes — you want deployments to be reproducible |
| Library (published for others to depend on) | No, by convention — let downstream users resolve their own compatible versions |

## A minimal working example: using `rand`

```toml
# Cargo.toml
[package]
name = "dice_roller"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
```

```rust
use rand::Rng;

fn roll_die() -> u32 {
    let mut rng = rand::thread_rng();
    rng.gen_range(1..=6)   // inclusive range -- 1 through 6
}

fn main() {
    let rolls: Vec<u32> = (0..5).map(|_| roll_die()).collect();
    println!("rolls: {:?}", rolls);

    let total: u32 = rolls.iter().sum();
    println!("total: {total}");
}
```

```text
rolls: [4, 1, 6, 3, 2]
total: 16
```

(The actual numbers will differ each run — that's the point of `rand`.)
Running `cargo build` the first time downloads and compiles `rand` and its
own dependencies automatically; every build after that reuses the cached,
already-compiled artifacts unless the source changes.

## Dev-dependencies: test-only crates

Some crates are only useful while developing or testing — a crate for
generating fake data in tests, for example — and don't need to ship inside
the final binary. `[dev-dependencies]` keeps them out of production builds
entirely:

```toml
[dependencies]
serde = "1.0"

[dev-dependencies]
# Only compiled for `cargo test` / `cargo bench`, never for `cargo build --release`
pretty_assertions = "1.4"
```

## Feature flags: opting into optional functionality

Crates often gate extra functionality (and extra compile-time cost) behind
**features**, so you only pay for what you use:

```toml
[dependencies]
# derive gives you #[derive(Serialize, Deserialize)]; without it,
# you'd have to hand-implement those traits yourself
serde = { version = "1.0", features = ["derive"] }

# default-features = false opts out of a crate's default feature set,
# useful for trimming compile time or binary size when you don't need everything
tokio = { version = "1", default-features = false, features = ["rt", "macros"] }
```

## Workspaces: multiple crates, one build

Once a project grows into several related crates (a core library plus a CLI
front-end plus a shared types crate, say), a **workspace** lets them share
one `Cargo.lock` and one `target/` build directory instead of rebuilding
shared dependencies redundantly for each crate.

```text
my_workspace/
    Cargo.toml          -- the workspace root, no [package] section
    core/
        Cargo.toml
        src/lib.rs
    cli/
        Cargo.toml       -- depends on `core` via a path dependency
        src/main.rs
```

```toml
# my_workspace/Cargo.toml (workspace root)
[workspace]
members = ["core", "cli"]
resolver = "2"
```

```toml
# my_workspace/cli/Cargo.toml
[package]
name = "cli"
version = "0.1.0"
edition = "2021"

[dependencies]
core = { path = "../core" }   # a path dependency -- points at a local crate, not crates.io
```

```bash
cargo build             # builds every member crate from the workspace root
cargo build -p cli       # builds just the `cli` member
cargo test               # runs tests across all members
```

## Cheat sheet

| Task | Command / syntax |
|------|-------------------|
| Add a dependency | `cargo add <crate>` or edit `[dependencies]` directly |
| Pin an exact version | `"=1.2.3"` |
| Allow compatible updates (default) | `"1.2.3"` (same as `"^1.2.3"`) |
| Update within allowed ranges | `cargo update` |
| Test-only dependency | `[dev-dependencies]` |
| Opt into optional functionality | `features = ["derive"]` |
| Reference a local crate | `{ path = "../other_crate" }` |
| Group multiple crates | `[workspace]` + `members = [...]` in the root `Cargo.toml` |

## Exercise

Create a new binary crate with `cargo new word_stats`. Add `rand = "0.8"` as
a dependency with `cargo add rand@0.8` (or by hand). Write a program that
builds a `Vec<&str>` of at least eight words, uses `rand::thread_rng()` and
`.gen_range()` to pick three random indices without repeats (a simple loop
with a `HashSet<usize>` to track which indices you've already used works
fine), and prints the three randomly chosen words. Run `cargo build` and
confirm `Cargo.lock` is created, then open it and find the line pinning the
exact resolved version of `rand`.

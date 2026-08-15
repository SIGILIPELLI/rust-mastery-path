# 09 · Workspace Organization

A single `Cargo.toml` works until a project grows a shared library, a CLI,
and an HTTP API that all need the same core types — at that point you want
one workspace with multiple crates, a single `Cargo.lock` shared across all
of them, and dependency versions declared once instead of copy-pasted per
crate. This module builds a real three-crate workspace: `core` (shared
types), `cli`, and `api`, both depending on `core`.

## Layout

```text
workspace-demo/
├── Cargo.toml           # [workspace] root, no [package]
└── crates/
    ├── core/
    │   ├── Cargo.toml
    │   └── src/lib.rs
    ├── cli/
    │   ├── Cargo.toml
    │   └── src/main.rs
    └── api/
        ├── Cargo.toml
        └── src/main.rs
```

## The workspace root

```toml
# Cargo.toml
[workspace]
resolver = "2"
members = ["crates/core", "crates/cli", "crates/api"]

[workspace.package]
version = "0.1.0"
edition = "2021"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
```

The root `Cargo.toml` has no `[package]` section — it's a workspace
manifest, not a crate itself. `[workspace.package]` and
`[workspace.dependencies]` are the two things member crates inherit from
instead of repeating: version numbers and shared dependency specs declared
once, at one version, for the whole workspace.

## `core` — the shared library crate

```toml
# crates/core/Cargo.toml
[package]
name = "core"
version.workspace = true
edition.workspace = true

[dependencies]
serde.workspace = true
```

```rust
// crates/core/src/lib.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    pub id: u32,
    pub title: String,
}

pub fn new_task(id: u32, title: &str) -> Task {
    Task { id, title: title.to_string() }
}
```

`version.workspace = true` and `serde.workspace = true` are the inheritance
syntax — this crate doesn't hardcode `"0.1.0"` or `serde = "1"` anywhere;
both come from the root. Bump `serde`'s version once in the workspace root
and every member that opted in with `.workspace = true` picks it up.

## `cli` and `api` — both depend on `core` by path

```toml
# crates/cli/Cargo.toml
[package]
name = "cli"
version.workspace = true
edition.workspace = true

[[bin]]
name = "tasks-cli"
path = "src/main.rs"

[dependencies]
core = { path = "../core" }
```

```rust
// crates/cli/src/main.rs
fn main() {
    let task = core::new_task(1, "write workspace docs");
    println!("{task:?}");
}
```

```toml
# crates/api/Cargo.toml
[dependencies]
core = { path = "../core" }
serde.workspace = true
```

`core = { path = "../core" }` is a path dependency, not a crates.io
version — Cargo resolves it to the sibling crate directly, so an edit to
`core`'s source is picked up by both `cli` and `api` the next time either
builds, no publishing or version bump required.

## Running it

```text
$ cargo run -p cli
Task { id: 1, title: "write workspace docs" }

$ cargo run -p api
{"id":2,"title":"serve tasks over http"}
```

`-p <crate>` selects which workspace member to run — from the workspace
root, plain `cargo run` is ambiguous once there's more than one binary and
Cargo will ask you to disambiguate.

## Seeing the shared dependency tree

```text
$ cargo tree -p cli
cli v0.1.0 (/workspace-demo/crates/cli)
└── core v0.1.0 (/workspace-demo/crates/core)
    └── serde v1.0.229
        ├── serde_core v1.0.229
        └── serde_derive v1.0.229 (proc-macro)
            ├── proc-macro2 v1.0.107
            │   └── unicode-ident v1.0.24
            ├── quote v1.0.47
            │   └── proc-macro2 v1.0.107 (*)
            └── syn v3.0.3
```

Only one `serde v1.0.229` appears anywhere in the tree, and only one
`Cargo.lock` exists at the workspace root — `cli` and `api` both depending
on `serde` (transitively through `core`, and directly in `api`'s case)
resolve to the *same* compiled version, compiled once and shared, instead of
each crate potentially locking a slightly different `serde` minor version
the way two independent projects could.

## Rust-specific traps

**One shared `Cargo.lock`, at the workspace root, not per crate.** Member
crates don't get their own lock file — `cargo build` from anywhere inside
the workspace resolves against (and updates) the single root
`Cargo.lock`. Committing a lock file inside `crates/core/` by accident (from
running `cargo build` there directly on an older Cargo, or from a stray
tool) is a common source of "why does CI show a diff in a file I never
touched."

**`resolver = "2"` isn't automatic on older `edition`.** Workspaces created
before the 2021 edition default to the old (version-1) feature resolver
unless `resolver = "2"` is set explicitly at the workspace root — the
version-1 resolver unifies feature flags across dev/build/normal
dependencies in a way that can silently enable a feature in a release build
that was only meant for tests. New workspaces created with a recent `edition`
get resolver 2 by default; older ones migrating need the line added by hand.

**Path dependencies don't imply a version bump obligation.** `core = {
path = "../core" }` compiles against whatever's on disk right now — there's
no separate "did I bump `core`'s version" step required for `cli` to see a
change, unlike depending on a crates.io-published version. This is
convenient during development and a real trap the moment `core` gets
published separately: forgetting to also bump `core`'s version for a
breaking change means downstream consumers pulling from crates.io (not the
workspace) silently get stale behavior.

**`cargo test` from the root runs every member's tests.** Plain `cargo
test` with no `-p` flag runs tests across the whole workspace — fine for CI,
surprising the first time a change to `api` triggers `core`'s and `cli`'s
test suites too because they're all one `cargo test` invocation.

## Cheat sheet

| Command | Effect |
|---|---|
| `cargo build` (from root) | Builds every workspace member |
| `cargo run -p cli` | Runs the `cli` member's binary specifically |
| `cargo test -p core` | Tests only the `core` member |
| `cargo tree -p cli` | Shows `cli`'s full dependency tree, deduped |
| `version.workspace = true` | Inherits `[workspace.package].version` |
| `dep.workspace = true` | Inherits the version/features from `[workspace.dependencies]` |
| `core = { path = "../core" }` | Local sibling-crate dependency, always current |

## Exercise

Add a fourth workspace member, `crates/shared-test-utils`, containing a
`pub fn sample_task() -> core::Task` helper, and add it as a
`[dev-dependencies]` entry (not a regular dependency) in both `cli` and
`api`'s `Cargo.toml`. Confirm with `cargo tree -p cli -e dev` that it shows
up only in the dev-dependency tree, and explain why a `[dev-dependencies]`
entry doesn't get compiled into `cli`'s release binary.

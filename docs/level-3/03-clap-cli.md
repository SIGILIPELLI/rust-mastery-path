# 03 · Building CLIs with clap

Every language eventually needs a story for command-line tools, and Rust's
is the `clap` crate. It parses `std::env::args()` into typed structs, derives
`--help` and `--version` for free, and turns "user typed something wrong"
into a clean error message instead of a panic. This module builds a small
task-tracker CLI with subcommands, flags, and defaults.

## Setup

```bash
cargo new tasks
cd tasks
cargo add clap --features derive
```

The `derive` feature is what lets you describe a CLI as a struct with
attributes, instead of hand-building a parser. It costs a bit of compile
time; it saves a lot of boilerplate.

## A struct is a CLI

```rust
use clap::{Parser, Subcommand};

/// A tiny task tracker
#[derive(Parser, Debug)]
#[command(name = "tasks", version, about)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand, Debug)]
enum Commands {
    /// Add a new task
    Add {
        text: String,
        #[arg(short, long, default_value_t = 3)]
        priority: u8,
    },
    /// List tasks
    List {
        #[arg(short, long)]
        verbose: bool,
    },
}

fn main() {
    let cli = Cli::parse();
    match cli.command {
        Commands::Add { text, priority } => {
            println!("added \"{text}\" with priority {priority}");
        }
        Commands::List { verbose } => {
            if verbose {
                println!("listing tasks (verbose)");
            } else {
                println!("listing tasks");
            }
        }
    }
}
```

`#[derive(Parser)]` on the top-level struct wires up `Cli::parse()`.
`#[command(subcommand)]` says "this field's variants become subcommands."
Each enum variant's fields become that subcommand's own arguments — `Add`
gets a positional `text` and an optional `--priority`, `List` gets a boolean
`--verbose` flag.

Running it:

```text
$ cargo run -q -- add "write docs" --priority 1
added "write docs" with priority 1

$ cargo run -q -- list -v
listing tasks (verbose)

$ cargo run -q -- --help
A tiny task tracker

Usage: tasks <COMMAND>

Commands:
  add   Add a new task
  list  List tasks
  help  Print this message or the help of the given subcommand(s)

Options:
  -h, --help     Print help
  -V, --version  Print version
```

That `--help` text was not written by hand — every line came from the doc
comments (`///`) and attributes on the struct and enum. Keep them accurate
and the help output stays accurate.

## Missing required arguments fail loudly, not silently

```text
$ cargo run -q -- add
error: the following required arguments were not provided:
  <TEXT>

Usage: tasks add <TEXT>

For more information, try '--help'.
```

Exit code is `2`. This is the behavior you want from a CLI: a clear message
to stderr, a nonzero exit code, no panic, no partial execution. You get it
without writing any validation code — `text: String` with no `Option`
wrapper and no default means clap treats it as required and enforces that
before your `main` body ever runs.

## Rust-specific traps

**Forgetting `derive` features.** `cargo add clap` alone gets you the
builder API, not the derive macros. If `#[derive(Parser)]` fails to compile
with "cannot find derive macro," check `Cargo.toml` for
`features = ["derive"]`.

**`String` vs `&str` in struct fields.** Derived CLI structs almost always
want owned `String` fields, not `&str`. Clap parses each argument into a new
owned value with no argv buffer to borrow from — trying to store `&'a str`
means fighting a lifetime clap doesn't hand you.

**Enum variant field shadowing.** Each `Commands` variant's fields are
scoped to that variant. It's tempting to hoist a shared flag like
`--verbose` onto the top-level `Cli` struct instead of duplicating it per
subcommand — usually the right call, but it changes where in the argv list
the flag has to appear (before vs. after the subcommand name), which trips
people testing by hand.

**`default_value_t` needs `Default`-free literal types.** `default_value_t
= 3` requires `priority: u8` to implement `Display` and be parseable from
a string via `FromStr` — works for numbers and `String` out of the box, but
a custom enum needs `#[derive(ValueEnum)]` or its own `FromStr` impl before
it can be a typed argument with a default.

## Cheat sheet

| Attribute | Effect |
|---|---|
| `#[derive(Parser)]` | Generates `parse()` / `try_parse()` on the struct |
| `#[command(subcommand)]` | Field's enum becomes the subcommand dispatch |
| `#[arg(short, long)]` | Enables both `-x` and `--xyz` forms |
| `#[arg(default_value_t = v)]` | Optional argument with a typed default |
| `Option<T>` field | Argument is optional, `None` if absent |
| `Vec<T>` field | Argument may be repeated / collects multiple values |
| `#[command(version, about)]` | Pulls `--version`/`--help` text from `Cargo.toml` / doc comments |

## Exercise

Extend the `tasks` CLI with a third subcommand, `Remove { id: u32 }`, and a
top-level `--json` flag (on `Cli`, not per-subcommand) that, when set,
changes every command's output to a JSON line instead of plain text — e.g.
`{"action":"add","text":"write docs","priority":1}`. Verify with
`cargo run -- --json add "ship it"` that the flag is readable from the
`Commands::Add` match arm even though it lives on the outer struct.

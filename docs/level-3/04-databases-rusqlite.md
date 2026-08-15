# 04 · Databases with rusqlite

`rusqlite` gives Rust a thin, typed wrapper over SQLite — no async runtime,
no server process, just a `.db` file (or `:memory:`) and a connection.
It's the fastest way to give a CLI tool or small service real persistence,
and it's a good place to see how Rust turns "the query might fail" and "the
row might not have the shape you expect" into values instead of exceptions.

## Setup

```bash
cargo new tasks-db
cd tasks-db
cargo add rusqlite --features bundled
```

`bundled` compiles SQLite itself into your binary instead of requiring the
system to have `libsqlite3` installed — slower first build, zero runtime
dependency problems. Drop the feature if you know the target already has
SQLite.

## Schema, insert, query

```rust
use rusqlite::{Connection, Result, params};

#[derive(Debug)]
struct Task {
    id: i64,
    title: String,
    done: bool,
}

fn main() -> Result<()> {
    let conn = Connection::open_in_memory()?;

    conn.execute(
        "CREATE TABLE tasks (
            id    INTEGER PRIMARY KEY,
            title TEXT NOT NULL,
            done  INTEGER NOT NULL DEFAULT 0
        )",
        (),
    )?;

    conn.execute(
        "INSERT INTO tasks (title, done) VALUES (?1, ?2)",
        params!["write docs", false],
    )?;
    conn.execute(
        "INSERT INTO tasks (title, done) VALUES (?1, ?2)",
        params!["ship release", true],
    )?;

    let mut stmt = conn.prepare("SELECT id, title, done FROM tasks ORDER BY id")?;
    let rows = stmt.query_map([], |row| {
        Ok(Task {
            id: row.get(0)?,
            title: row.get(1)?,
            done: row.get::<_, i64>(2)? != 0,
        })
    })?;

    for task in rows {
        let task = task?;
        println!("{:?}", task);
    }

    Ok(())
}
```

```text
Task { id: 1, title: "write docs", done: false }
Task { id: 2, title: "ship release", done: true }
```

`main() -> Result<()>` plus `?` on every fallible call is the idiomatic
shape for a small program that talks to a database: any `rusqlite::Error`
(malformed SQL, a broken connection, a constraint violation) short-circuits
out of `main` and prints via `Debug`, instead of you writing `.unwrap()`
everywhere and panicking on the first bad row.

`row.get::<_, i64>(2)?` is an explicit turbofish because SQLite has no
native boolean type — it stores `0`/`1` as an integer, so you pull an `i64`
out and convert. This is a small but real trap: `row.get::<_, bool>(2)`
compiles and even works on some SQLite builds, but relying on it is fragile
across schema versions that stored booleans differently.

## Failure is a value, not an exception

```rust
let bad = conn.execute("INSERT INTO tasks (title) VALUES (NULL)", ());
match bad {
    Ok(_) => println!("unexpectedly succeeded"),
    Err(e) => println!("insert failed as expected: {e}"),
}
```

```text
insert failed as expected: NOT NULL constraint failed: tasks.title
```

The `NOT NULL` constraint lives in the schema, SQLite enforces it, and
`rusqlite` surfaces the failure as `Err(rusqlite::Error::SqliteFailure(...))`
with a human-readable message via `Display`. Nothing unwound the stack —
you get to decide whether that's a fatal error, a retry, or a message back
to a caller.

## Rust-specific traps

**`Connection` is not `Sync`.** A single `rusqlite::Connection` cannot be
shared across threads by reference; SQLite's C API isn't safe for
concurrent use from one handle. The common fix is `Arc<Mutex<Connection>>`,
or better, a connection pool (`r2d2_sqlite`) that hands each thread its own
connection.

**Borrowed rows don't outlive the statement.** `stmt.query_map` returns an
iterator borrowing from `stmt`; you cannot return that iterator from a
function while dropping `stmt`, and the borrow checker will say so plainly.
Collect into a `Vec<Task>` before the statement goes out of scope if the
rows need to escape the function.

**`params!` macro vs. tuples.** `conn.execute(sql, (a, b))` and
`conn.execute(sql, params![a, b])` look interchangeable but differ once
types get involved — `params!` will happily coerce a `&str` and a `String`
side by side, plain tuples are pickier about matching `ToSql` impls
consistently. When a query's parameter types are mixed, reach for `params!`
first.

**Named vs. positional placeholders.** Mixing `?1`, `?2` (positional) with
`:name` (named) placeholders in the same statement is legal SQL but a
frequent source of "wrong value in wrong column" bugs that the compiler
cannot catch — SQLite only tells you at query-prepare time, and only if the
name doesn't match at all.

## Cheat sheet

| Call | Purpose |
|---|---|
| `Connection::open("file.db")` | Open (creating if absent) a database file |
| `Connection::open_in_memory()` | Ephemeral `:memory:` database, useful for tests |
| `conn.execute(sql, params)` | Run INSERT/UPDATE/DELETE/DDL, returns rows affected |
| `conn.prepare(sql)` | Compile a statement for reuse or row iteration |
| `stmt.query_map(params, closure)` | Row iterator, closure maps each row to a value |
| `row.get::<_, T>(i)` | Typed column extraction by index, fallible |
| `params![a, b, ...]` | Macro for building a heterogeneous parameter list |

## Exercise

Add an `UPDATE tasks SET done = 1 WHERE id = ?1` call that marks task `1`
done, then re-run the `SELECT` query and confirm the printed `Task` struct
shows `done: true`. Then try inserting a task with an `id` that already
exists (`INSERT INTO tasks (id, title) VALUES (1, 'dup')`) and print the
`rusqlite::Error` you get back — identify which SQLite constraint it names.

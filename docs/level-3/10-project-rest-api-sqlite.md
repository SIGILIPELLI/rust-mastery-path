# 10 · Project: REST API with SQLite

This project combines the last three modules — axum, rusqlite, and the
error-handling patterns from earlier levels — into a small persistent
book-catalog API: `GET`/`POST /books`, `GET`/`DELETE /books/{id}`, backed by
a real SQLite file. It's structured as a small multi-file crate instead of
one `main.rs`, which is the shape any real axum + rusqlite service ends up
taking.

## Project layout

```text
rest-api-sqlite/
├── Cargo.toml
└── src/
    ├── main.rs      # wiring: router, shared state, server startup
    ├── models.rs     # Book / NewBook structs
    ├── db.rs         # all SQL lives here
    └── handlers.rs   # axum handlers, translate db results to HTTP
```

```bash
cargo new rest-api-sqlite
cd rest-api-sqlite
cargo add axum tokio --features tokio/full
cargo add serde --features derive
cargo add rusqlite --features bundled
```

## `src/models.rs` — the data shapes

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Book {
    pub id: i64,
    pub title: String,
    pub author: String,
    pub year: i32,
}

#[derive(Debug, Deserialize)]
pub struct NewBook {
    pub title: String,
    pub author: String,
    pub year: i32,
}
```

Two structs, not one: `Book` has an `id` because it exists in the database;
`NewBook` doesn't, because the client isn't supposed to choose the id. This
split is the whole reason "add an id field with a default" isn't a good
enough substitute — a `NewBook` with an `id` field would let a client
overwrite arbitrary rows via a crafted request body.

## `src/db.rs` — all SQL in one place

```rust
use rusqlite::{Connection, Result, params};
use crate::models::{Book, NewBook};

pub fn init(conn: &Connection) -> Result<()> {
    conn.execute(
        "CREATE TABLE IF NOT EXISTS books (
            id     INTEGER PRIMARY KEY AUTOINCREMENT,
            title  TEXT NOT NULL,
            author TEXT NOT NULL,
            year   INTEGER NOT NULL
        )",
        (),
    )?;
    Ok(())
}

pub fn insert_book(conn: &Connection, new: &NewBook) -> Result<Book> {
    conn.execute(
        "INSERT INTO books (title, author, year) VALUES (?1, ?2, ?3)",
        params![new.title, new.author, new.year],
    )?;
    let id = conn.last_insert_rowid();
    Ok(Book { id, title: new.title.clone(), author: new.author.clone(), year: new.year })
}

pub fn list_books(conn: &Connection) -> Result<Vec<Book>> {
    let mut stmt = conn.prepare("SELECT id, title, author, year FROM books ORDER BY id")?;
    let rows = stmt.query_map([], |row| {
        Ok(Book { id: row.get(0)?, title: row.get(1)?, author: row.get(2)?, year: row.get(3)? })
    })?;
    rows.collect()
}

pub fn get_book(conn: &Connection, id: i64) -> Result<Option<Book>> {
    conn.query_row(
        "SELECT id, title, author, year FROM books WHERE id = ?1",
        params![id],
        |row| Ok(Book { id: row.get(0)?, title: row.get(1)?, author: row.get(2)?, year: row.get(3)? }),
    )
    .map(Some)
    .or_else(|e| match e {
        rusqlite::Error::QueryReturnedNoRows => Ok(None),
        other => Err(other),
    })
}

pub fn delete_book(conn: &Connection, id: i64) -> Result<bool> {
    let affected = conn.execute("DELETE FROM books WHERE id = ?1", params![id])?;
    Ok(affected > 0)
}
```

`get_book` returns `Result<Option<Book>, Error>` — three real outcomes,
three real types: a genuine database error, a clean "not found," and
"found." `query_row` treats zero rows as an `Err(QueryReturnedNoRows)`, not
an `Ok(None)`, so this function's job is converting that specific error
variant into `Ok(None)` while letting every other error still propagate as
an actual error. Collapsing all of that into a single `Option<Book>` (Ok/Err
folded together) would hide real database failures behind "not found."

## `src/handlers.rs` — HTTP glue, no SQL

```rust
use axum::{extract::{Path, State}, http::StatusCode, Json};
use crate::{db, models::{Book, NewBook}, AppState};

pub async fn list_books(State(state): State<AppState>) -> Result<Json<Vec<Book>>, StatusCode> {
    let conn = state.conn.lock().unwrap();
    db::list_books(&conn).map(Json).map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)
}

pub async fn create_book(
    State(state): State<AppState>,
    Json(new): Json<NewBook>,
) -> Result<(StatusCode, Json<Book>), StatusCode> {
    let conn = state.conn.lock().unwrap();
    db::insert_book(&conn, &new)
        .map(|b| (StatusCode::CREATED, Json(b)))
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)
}

pub async fn get_book(State(state): State<AppState>, Path(id): Path<i64>) -> Result<Json<Book>, StatusCode> {
    let conn = state.conn.lock().unwrap();
    match db::get_book(&conn, id) {
        Ok(Some(book)) => Ok(Json(book)),
        Ok(None) => Err(StatusCode::NOT_FOUND),
        Err(_) => Err(StatusCode::INTERNAL_SERVER_ERROR),
    }
}

pub async fn delete_book(State(state): State<AppState>, Path(id): Path<i64>) -> StatusCode {
    let conn = state.conn.lock().unwrap();
    match db::delete_book(&conn, id) {
        Ok(true) => StatusCode::NO_CONTENT,
        Ok(false) => StatusCode::NOT_FOUND,
        Err(_) => StatusCode::INTERNAL_SERVER_ERROR,
    }
}
```

Every handler acquires the lock, does exactly one `db::` call, and drops the
lock when the function returns — never across an `.await`. That's the same
constraint from the axum module, now applied to a real multi-handler
service where it's easier to violate by accident (e.g. locking, then
awaiting a second query before releasing).

## `src/main.rs` — wiring

```rust
mod db;
mod handlers;
mod models;

use axum::{routing::get, Router};
use rusqlite::Connection;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
pub struct AppState {
    pub conn: Arc<Mutex<Connection>>,
}

#[tokio::main]
async fn main() {
    let conn = Connection::open("books.db").expect("open db");
    db::init(&conn).expect("init schema");
    let state = AppState { conn: Arc::new(Mutex::new(conn)) };

    let app = Router::new()
        .route("/books", get(handlers::list_books).post(handlers::create_book))
        .route("/books/{id}", get(handlers::get_book).delete(handlers::delete_book))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3002").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

`Connection::open("books.db")` (not `:memory:`) means the catalog survives
restarts — a real file sits next to the binary. `AppState` wraps it in
`Arc<Mutex<_>>` for the same reason the axum module did: one SQLite
connection, shared, one request touching it at a time.

## Running it

```text
$ cargo run
listening on 127.0.0.1:3002
```

```text
$ curl -s -X POST localhost:3002/books -H 'content-type: application/json' \
    -d '{"title":"The Rust Book","author":"Steve K.","year":2019}'
{"id":1,"title":"The Rust Book","author":"Steve K.","year":2019}

$ curl -s -X POST localhost:3002/books -H 'content-type: application/json' \
    -d '{"title":"Programming Rust","author":"Jim Blandy","year":2021}'
{"id":2,"title":"Programming Rust","author":"Jim Blandy","year":2021}

$ curl -s localhost:3002/books
[{"id":1,"title":"The Rust Book","author":"Steve K.","year":2019},{"id":2,"title":"Programming Rust","author":"Jim Blandy","year":2021}]

$ curl -s localhost:3002/books/1
{"id":1,"title":"The Rust Book","author":"Steve K.","year":2019}

$ curl -s -o /dev/null -w "DELETE status=%{http_code}\n" -X DELETE localhost:3002/books/1
DELETE status=204

$ curl -s -o /dev/null -w "GET after delete status=%{http_code}\n" localhost:3002/books/1
GET after delete status=404

$ curl -s localhost:3002/books
[{"id":2,"title":"Programming Rust","author":"Jim Blandy","year":2021}]
```

The delete-then-get sequence is the real proof this is a persistent,
correctly-wired system, not a happy-path demo: the row is gone from both the
list and the direct lookup, and the second `GET` on a deleted id correctly
reports `404`, not a stale `200` or a crash.

## Rust-specific traps

**Module privacy across files.** `mod db;` in `main.rs` only makes `db`'s
`pub` items visible; forgetting `pub` on `insert_book` or on `Book`'s fields
produces "private field" errors from `handlers.rs` that look like a
different problem (often mistaken for a missing import).

**Splitting `AppState` across files creates an import cycle risk.**
`handlers.rs` needs `AppState` from `main.rs` (`use crate::AppState`), and
`main.rs` needs `handlers::` functions — this works because Rust resolves
module trees before checking, but reorganizing later (e.g. moving
`AppState` into its own file) requires updating every `use crate::AppState`
site consistently, and the compiler only tells you one broken import at a
time.

**Bundled SQLite means a real (slow) first build.** `--features bundled`
compiles the SQLite C library from source on first `cargo build`; subsequent
builds are fast because it's cached, but a `cargo clean` resets that cost —
worth knowing before assuming a slow CI run is a caching bug.

## Stretch goals

- Add `PUT /books/{id}` to update an existing book's fields, returning `200`
  with the updated `Book` or `404` if the id doesn't exist.
- Add a `?year=` query parameter to `GET /books` (via axum's `Query`
  extractor) that filters the list to books published in that year.
- Replace the raw SQL strings in `db.rs` with `rusqlite::named_params!` and
  compare readability on the four-column `INSERT`.
- Swap `std::sync::Mutex<Connection>` for an `r2d2_sqlite` connection pool so
  concurrent requests aren't serialized on a single lock, and measure the
  difference under a quick `wrk` or `hey` load test.

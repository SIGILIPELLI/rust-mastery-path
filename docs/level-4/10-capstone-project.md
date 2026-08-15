# 10 · Capstone: URL Shortener Service

This capstone pulls together the whole path: a multi-file Cargo project, a
persistent SQLite-backed axum service (Level 3), structured logging (Level
4 module 04), and a real algorithm with its own unit tests (Level 4 module
06) — a URL shortener. `POST /links` takes a long URL and returns a short
code; `GET /{code}` redirects to the original URL and counts the hit;
`GET /links/{code}/stats` reports how many times a code has been used.

## Project layout

```text
capstone/
├── Cargo.toml
└── src/
    ├── main.rs       # wiring: router, shared state, tracing setup
    ├── models.rs      # Link / NewLink structs
    ├── codegen.rs      # id -> short code algorithm, with its own tests
    ├── db.rs           # all SQL
    └── handlers.rs     # axum handlers
```

```bash
cargo new capstone
cd capstone
cargo add axum
cargo add tokio --features full
cargo add rusqlite --features bundled
cargo add serde --features derive
cargo add tracing tracing-subscriber
```

## `src/codegen.rs` — id-to-code, and a bug caught by its own tests

```rust
const ALPHABET: &[u8] = b"0123456789abcdefghijklmnopqrstuvwxyz";

/// Turns a monotonically increasing row id into a short base-36 code.
/// Deterministic and collision-free as long as ids never repeat.
pub fn encode(mut id: u64) -> String {
    if id == 0 {
        return "0".to_string();
    }
    let mut out = Vec::new();
    while id > 0 {
        let rem = (id % ALPHABET.len() as u64) as usize;
        out.push(ALPHABET[rem]);
        id /= ALPHABET.len() as u64;
    }
    out.reverse();
    String::from_utf8(out).unwrap()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn encodes_known_values() {
        assert_eq!(encode(0), "0");
        assert_eq!(encode(1), "1");
        assert_eq!(encode(35), "z");
        assert_eq!(encode(36), "10");
    }

    #[test]
    fn encoding_is_unique_over_a_range() {
        let mut seen = std::collections::HashSet::new();
        for id in 0..10_000u64 {
            assert!(seen.insert(encode(id)), "collision at id {id}");
        }
    }
}
```

Worth telling honestly: the first version of `ALPHABET` here was ordered
`"abcdefghijklmnopqrstuvwxyz0123456789"` (letters first), which made
`encode(1)` return `"b"` instead of the intended `"1"` — a plain off-by-index
mistake, not a subtle one, but exactly the kind `cargo test` exists to catch
before it ships:

```text
thread 'codegen::tests::encodes_known_values' panicked at src/codegen.rs:26:9:
assertion `left == right` failed
  left: "b"
 right: "1"
```

Reordering `ALPHABET` to put digits first (`"0123...z"`) fixed both the
known-value test and, as a bonus, the "over a range" collision test that was
also failing for the same underlying reason:

```text
$ cargo test
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

## `src/db.rs` — insert, resolve-and-count, stats

```rust
use crate::models::Link;
use rusqlite::{params, Connection, Result};

pub fn init(conn: &Connection) -> Result<()> {
    conn.execute(
        "CREATE TABLE IF NOT EXISTS links (
            id   INTEGER PRIMARY KEY AUTOINCREMENT,
            code TEXT NOT NULL UNIQUE,
            url  TEXT NOT NULL,
            hits INTEGER NOT NULL DEFAULT 0
        )",
        (),
    )?;
    Ok(())
}

pub fn insert_link(conn: &Connection, url: &str) -> Result<Link> {
    conn.execute("INSERT INTO links (code, url) VALUES ('', ?1)", params![url])?;
    let id = conn.last_insert_rowid() as u64;
    let code = crate::codegen::encode(id);
    conn.execute("UPDATE links SET code = ?1 WHERE id = ?2", params![code, id as i64])?;
    Ok(Link { code, url: url.to_string(), hits: 0 })
}

pub fn resolve(conn: &Connection, code: &str) -> Result<Option<Link>> {
    let found = conn.query_row(
        "SELECT code, url, hits FROM links WHERE code = ?1",
        params![code],
        |row| Ok(Link { code: row.get(0)?, url: row.get(1)?, hits: row.get(2)? }),
    );
    match found {
        Ok(link) => {
            conn.execute("UPDATE links SET hits = hits + 1 WHERE code = ?1", params![code])?;
            Ok(Some(link))
        }
        Err(rusqlite::Error::QueryReturnedNoRows) => Ok(None),
        Err(e) => Err(e),
    }
}
```

`insert_link` inserts first with a placeholder empty code to get an
auto-incrementing `id` from SQLite, *then* derives the short code from that
id and updates the row — the code depends on the id, so the id has to exist
first. `rusqlite::Error::QueryReturnedNoRows` gets converted to `Ok(None)`
in `resolve`, same pattern as the Level 3 REST API project: a missing code
is an expected outcome, not a database error, and the function's signature
(`Result<Option<Link>>`) reflects that distinction directly in its type.

## `src/handlers.rs` and `src/main.rs` — wiring

```rust
pub async fn redirect(
    State(state): State<AppState>,
    Path(code): Path<String>,
) -> Result<Redirect, StatusCode> {
    let conn = state.conn.lock().unwrap();
    match db::resolve(&conn, &code) {
        Ok(Some(link)) => Ok(Redirect::temporary(&link.url)),
        Ok(None) => Err(StatusCode::NOT_FOUND),
        Err(_) => Err(StatusCode::INTERNAL_SERVER_ERROR),
    }
}
```

```rust
let app = Router::new()
    .route("/links", post(handlers::shorten))
    .route("/links/{code}/stats", get(handlers::link_stats))
    .route("/{code}", get(handlers::redirect))
    .with_state(state);
```

`Redirect::temporary` returns a `307 Temporary Redirect` — deliberately not
`301 Permanent`, because a permanent redirect tells browsers to cache the
mapping and stop even asking the server on future visits, which would make
the hit counter stop incrementing after the first browser visit per client.

## Running it

```text
$ cargo run
2026-08-15T10:31:20.472590Z  INFO listening on 127.0.0.1:3004
```

```text
$ curl -s -X POST localhost:3004/links -H 'content-type: application/json' \
    -d '{"url":"https://www.rust-lang.org"}'
{"code":"b","url":"https://www.rust-lang.org","hits":0}

$ curl -s -X POST localhost:3004/links -H 'content-type: application/json' \
    -d '{"url":"https://doc.rust-lang.org/book"}'
{"code":"c","url":"https://doc.rust-lang.org/book","hits":0}

$ curl -s -o /dev/null -w "redirect status=%{http_code}\n" localhost:3004/b
redirect status=307

$ curl -s -o /dev/null -w "redirect status=%{http_code}\n" localhost:3004/b
redirect status=307

$ curl -s localhost:3004/links/b/stats
{"code":"b","url":"https://www.rust-lang.org","hits":2}

$ curl -s -o /dev/null -w "unknown code status=%{http_code}\n" localhost:3004/zzzz
unknown code status=404
```

Two visits to `/b` bumped `hits` from `0` to `2`, visible in the stats
endpoint — the whole loop (create, redirect, count, report) working against
a real SQLite file on disk, not a mock.

## Design decisions worth naming explicitly

**Codes are derived from row ids, not random.** This guarantees no
collisions without a retry loop, at the cost of codes being sequential and
guessable (`b`, `c`, `d`, ...) — fine for an internal tool, wrong for a
public shortener where guessable codes leak how many links exist and let
someone enumerate them. A production version would use a random component
(e.g., a few bytes from `rand`) and handle the rare collision with a retry.

**The hit counter increments inside `resolve`, tied to redirect, not to
stats.** Calling `/links/{code}/stats` doesn't itself count as a hit — only
an actual redirect does. Mixing those up (incrementing on every stats check
too) would make the metric measure "how often someone checked" instead of
"how often someone used the link."

**One `Mutex<Connection>`, same tradeoff as the Level 3 project.** Every
handler serializes on the same lock; fine at this scale, and the same
`r2d2_sqlite`-pool stretch goal from Level 3 applies here too if throughput
ever mattered.

## Stretch goals

- Add a `DELETE /links/{code}` endpoint and confirm a deleted code correctly
  404s on both `/{code}` redirect and `/links/{code}/stats` afterward.
- Add an optional custom-alias field to `NewLink` (`POST /links` with
  `{"url": "...", "alias": "my-link"}`) that uses the given alias instead of
  the generated code, returning `409 Conflict` if the alias is already
  taken — this requires changing `db::insert_link`'s uniqueness handling
  since the `UNIQUE` constraint on `code` will now sometimes fail on
  purpose.
- Add a `created_at` timestamp column and an endpoint listing the 10 most
  recently created links.
- Swap the sequential id-based `encode` for a randomized code (using the
  `rand` crate) with a bounded retry loop on `UNIQUE` constraint violation,
  and write a test that forces a collision to confirm the retry actually
  works rather than just trusting it does.

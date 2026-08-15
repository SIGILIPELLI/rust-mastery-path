# 01 · Advanced Async Patterns

Basic `async`/`.await` gets one future running at a time, sequentially.
Real async code needs to run several futures concurrently, race them against
each other, cancel work that's no longer needed, and bound how long
something is allowed to take. Tokio gives you four primitives for this —
`join!`, `select!`, task cancellation, and `timeout` — and each one has a
sharp edge specific to how Rust futures actually work.

## `join!`: run several futures concurrently, wait for all

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn fetch(id: u32, delay_ms: u64) -> String {
    sleep(Duration::from_millis(delay_ms)).await;
    format!("resource {id}")
}

async fn joined_example() {
    let start = std::time::Instant::now();
    let (a, b, c) = tokio::join!(fetch(1, 100), fetch(2, 50), fetch(3, 150));
    println!("{a} {b} {c} in {:?}", start.elapsed());
}
```

```text
resource 1 resource 2 resource 3 in 152.091584ms
```

The three fetches take 100ms, 50ms, and 150ms — run sequentially that's
300ms; `join!` runs them concurrently on the same task, so the total is
bounded by the slowest one (~150ms), not the sum. `join!` polls all three
futures on every wakeup, driving whichever ones are ready, and only returns
once every future has completed. Unlike `tokio::spawn`, this concurrency
doesn't need a separate OS thread or task — it's all interleaved polling
inside one `async fn`.

## `select!`: race futures, take the first to finish

```rust
async fn select_example() {
    let winner = tokio::select! {
        r = fetch(1, 30) => format!("fast: {r}"),
        r = fetch(2, 200) => format!("slow: {r}"),
    };
    println!("select winner: {winner}");
}
```

```text
select winner: fast: resource 1
```

`select!` polls every branch and, the moment one completes, returns its
value and **drops the rest**. That drop is the sharp edge: the `fetch(2,
200)` future is dropped mid-flight, not paused and resumable. If that
future had, say, half-written a file or held a lock past an `.await` point,
`select!` doesn't clean that up for you — the future's own `Drop` impl (if
any) is all that runs.

## Cancellation: `spawn` + `abort` doesn't mean "stops immediately"

```rust
async fn cancellation_example() {
    let handle = tokio::spawn(async {
        for i in 0..5 {
            sleep(Duration::from_millis(50)).await;
            println!("  tick {i}");
        }
        "finished"
    });

    sleep(Duration::from_millis(120)).await;
    handle.abort();

    match handle.await {
        Ok(v) => println!("task completed: {v}"),
        Err(e) if e.is_cancelled() => println!("task was cancelled"),
        Err(e) => println!("task panicked: {e}"),
    }
}
```

```text
  tick 0
  tick 1
task was cancelled
```

Two ticks print (at ~50ms and ~100ms) before `abort()` fires at 120ms — the
task was mid-`sleep` on its third iteration when cancellation landed.
`abort()` doesn't run any more of the task's code; it marks the task for
cancellation at its *next* `.await` point, and the awaited `JoinHandle`
resolves to `Err(JoinError)` with `.is_cancelled() == true`. Cancellation in
async Rust is cooperative in this specific sense — it happens at await
points, not instantly at the moment `abort()` is called.

## `timeout`: bound how long a future is allowed to run

```rust
async fn timeout_example() {
    let result = tokio::time::timeout(Duration::from_millis(50), fetch(1, 200)).await;
    match result {
        Ok(v) => println!("got {v} in time"),
        Err(_) => println!("timed out waiting for fetch"),
    }
}
```

```text
timed out waiting for fetch
```

`timeout` wraps a future and races it against a timer, exactly the shape
`select!` has under the hood. The 200ms fetch never gets to finish because
the 50ms timer wins — and the fetch future is dropped, same caveat as
`select!`: nothing partial it did gets undone automatically.

## Rust-specific traps

**Cancellation drops futures, it doesn't run their "rest."** A future
that's midway through, say, `write_all().await` when cancelled doesn't
finish the write and doesn't roll it back — it just stops being polled and
gets dropped. Any cleanup has to be encoded in `Drop`, or the operation has
to be structured so partial progress is safe (e.g. writing to a temp file
and renaming atomically at the end, not the moment before cancellation
could land).

**Borrowing across `select!` branches.** Each branch in a `select!` block
can't hold a mutable borrow that another branch also needs, because the
compiler has to prove the dropped branches don't leave dangling borrows
behind. This produces borrow-checker errors that are hard to read because
they point at the `select!` macro expansion, not your code directly.

**`join!` isn't `try_join!`.** `join!` waits for every future regardless of
whether earlier ones returned an `Err` — if `fetch` returned
`Result<String, E>`, a fast failure in the first future doesn't short-circuit
the other two. `tokio::try_join!` is the version that returns early on the
first `Err` and cancels the rest, which is usually what "concurrent, but
fail fast" actually wants.

**Spawned tasks outlive their creating function unless awaited or aborted.**
`tokio::spawn` detaches immediately — if `cancellation_example` returned
without ever calling `.abort()` or `.await`ing the handle, the spawned task
would keep running in the background on the runtime, invisible to anyone,
until it finished or the whole runtime shut down.

## Cheat sheet

| Primitive | Behavior |
|---|---|
| `tokio::join!(a, b, c)` | Runs all concurrently, waits for all, keeps every result |
| `tokio::try_join!(a, b, c)` | Like `join!`, but short-circuits and cancels the rest on first `Err` |
| `tokio::select! { .. }` | Races branches, takes first ready, drops the others |
| `handle.abort()` | Requests cancellation at the task's next `.await` point |
| `tokio::time::timeout(d, fut)` | Races `fut` against a timer, `Err` on timeout |
| `JoinError::is_cancelled()` | True if a joined task was aborted rather than panicked |

## Exercise

Write an async function `fetch_with_retry(id: u32, delay_ms: u64, attempts:
u32)` that calls `fetch`, wraps each attempt in a 100ms `timeout`, and on
timeout retries up to `attempts` times before giving up and returning
`Err(String)`. Call it with a `fetch` that's slower than 100ms and confirm
it exhausts all attempts and returns `Err`; then call it with a fetch under
100ms and confirm it succeeds on the first attempt.

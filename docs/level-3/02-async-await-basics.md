# 02 · Async/Await Basics

[Module 01](01-concurrency.md) covered OS threads — real parallelism, one
stack per thread, managed by the operating system's scheduler. Async Rust
solves a different problem: running huge numbers of tasks that spend most of
their time *waiting* (on a socket, a timer, a database) without paying for a
thread per task. An `async fn` compiles down to a state machine that can be
paused at each `.await` point and resumed later — Rust's standard library
defines the `Future` trait and the `async`/`await` syntax, but *executing*
futures requires a runtime. This module uses [tokio](https://tokio.rs), the
most widely used one.

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

## `async fn` and `.await`

```rust
use std::time::Duration;

async fn fetch(name: &str, ms: u64) -> String {
    tokio::time::sleep(Duration::from_millis(ms)).await;
    format!("{name} done")
}

#[tokio::main]
async fn main() {
    let result = fetch("job", 50).await;
    println!("{result}");
}
```

```text
job done
```

`async fn fetch(..) -> String` doesn't actually return a `String` when
called — it returns a value implementing `Future<Output = String>` that does
nothing until polled. `.await` is what polls it (and yields control back to
the runtime while it's waiting). `#[tokio::main]` is a macro that wraps
`main` in the boilerplate needed to create a tokio runtime and block on the
async `main` body — nothing runs asynchronously without a runtime driving
it, which is the first surprise for people coming from JavaScript, where an
event loop is always implicitly present.

## Sequential `.await` vs concurrent futures

Awaiting one future after another doesn't overlap their waiting time —
that's easy to miss, because the syntax reads like it should:

```rust
use std::time::{Duration, Instant};

async fn fetch(name: &str, ms: u64) -> String {
    tokio::time::sleep(Duration::from_millis(ms)).await;
    format!("{name} done")
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let a = fetch("a", 50).await;
    let b = fetch("b", 50).await;
    println!("{a}, {b}");
    println!("sequential: {:?}", start.elapsed());

    let start = Instant::now();
    let (a, b) = tokio::join!(fetch("a", 50), fetch("b", 50));
    println!("{a}, {b}");
    println!("concurrent: {:?}", start.elapsed());
}
```

```text
a done, b done
sequential: 104.159917ms
a done, b done
concurrent: 52.129208ms
```

`a.await` fully completes before `fetch("b", ..)` is even created, so the two
50ms sleeps add up. `tokio::join!` polls both futures on the same task,
switching between them whenever one is waiting — so the two sleeps overlap
and the total is roughly the *longest* one, not the sum. Reaching for
`join!` (or `tokio::spawn`, below) instead of back-to-back `.await` is the
single most common fix for a slow async program that "should" be concurrent.

## The trap: blocking the executor

```rust
use std::time::{Duration, Instant};

async fn blocking_task(name: &str) -> String {
    std::thread::sleep(Duration::from_millis(50)); // WRONG inside async code
    format!("{name} done")
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let (a, b) = tokio::join!(blocking_task("a"), blocking_task("b"));
    println!("{a}, {b}");
    println!("blocking join: {:?}", start.elapsed());
}
```

```text
a done, b done
blocking join: 106.340333ms
```

This compiles fine — `std::thread::sleep` isn't an `.await` point, so
nothing here is technically wrong from the type checker's point of view. But
the timing gives it away: it took ~106ms, not ~52ms, even though we used the
same `join!` as the concurrent example above. `std::thread::sleep` blocks
the *entire OS thread* the task happens to be running on, so the runtime
can't switch to the other future while it waits — the two "concurrent"
tasks end up running one after another anyway. Any blocking call (sleeping,
synchronous file I/O, a CPU-heavy loop) inside an `async fn` has this same
effect. The fix is either an async-aware equivalent (`tokio::time::sleep`,
`tokio::fs`) or, for unavoidable blocking work, `tokio::task::spawn_blocking`
to move it onto a dedicated thread pool.

## `tokio::spawn` — running a task in the background

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        for i in 1..=3 {
            tokio::time::sleep(Duration::from_millis(10)).await;
            println!("spawned: {i}");
        }
        "spawned task result"
    });

    println!("main continues");
    let result = handle.await.unwrap();
    println!("{result}");
}
```

```text
main continues
spawned: 1
spawned: 2
spawned: 3
spawned task result
```

`tokio::spawn` hands the future to the runtime immediately and returns a
`JoinHandle`, much like `thread::spawn` returns a `JoinHandle` in Module 01
— and just like that thread version, the spawned future must be `'static`
(own everything it touches, typically via `move`) because the runtime may
run it long after the calling scope ends. Awaiting the handle blocks *this*
task until the spawned one finishes and yields a `Result` (`Err` if the
spawned task panicked), mirroring `.join().unwrap()` almost exactly.

## The Rust-specific trap: cancellation

This one has no analogue in synchronous Rust. Dropping a future stops it
*wherever it happens to be paused* — there's no "finally" block, no
guarantee the rest of the function body ever runs:

```rust
use std::time::Duration;

async fn slow_write() {
    for step in 1..=5 {
        tokio::time::sleep(Duration::from_millis(20)).await;
        println!("  slow_write: step {step} committed");
    }
    println!("  slow_write: finished cleanly");
}

#[tokio::main]
async fn main() {
    println!("racing slow_write against a 50ms timeout");
    tokio::select! {
        _ = slow_write() => {
            println!("slow_write won the race");
        }
        _ = tokio::time::sleep(Duration::from_millis(50)) => {
            println!("timeout won the race -- slow_write is dropped here, mid-step");
        }
    }
    println!("main exits");
}
```

```text
racing slow_write against a 50ms timeout
  slow_write: step 1 committed
  slow_write: step 2 committed
timeout won the race -- slow_write is dropped here, mid-step
main exits
```

`tokio::select!` runs both branches concurrently and takes whichever
finishes first, dropping the other. Here the timeout wins after 50ms, and
`slow_write` — mid-loop, having only printed steps 1 and 2 — is simply
dropped. Its "step 3" print and its "finished cleanly" line never happen; no
panic, no error, the code after the cancelled point silently never runs.
This is why async code that must clean up after itself (releasing a lock,
finishing a partial write, decrementing a counter) needs to structure that
cleanup so it survives being dropped mid-await — typically via a type whose
`Drop` impl does the cleanup synchronously, since `Drop::drop` cannot itself
be async. Reaching for `tokio::spawn` plus `JoinHandle::abort()` when you
need cancellation *with* an explicit signal, instead of implicit
drop-on-select, is often clearer than relying on this behavior by accident.

## Cheat sheet

| Tool | Use when |
|------|----------|
| `async fn` | Declare a function whose body can pause at `.await` points |
| `.await` | Poll a future, yielding control back to the runtime while it waits |
| `#[tokio::main]` | Set up a tokio runtime and run an async `main` |
| `tokio::join!(a, b)` | Run multiple futures concurrently, wait for all of them |
| `tokio::spawn(fut)` | Hand a `'static` future to the runtime to run in the background |
| `handle.await` | Wait for a spawned task, getting a `Result` (`Err` on panic) |
| `tokio::select!` | Race futures; the first to finish wins, the rest are dropped |
| `tokio::time::sleep` | Async-friendly sleep (unlike `std::thread::sleep`, doesn't block the executor) |
| `tokio::task::spawn_blocking` | Move genuinely blocking work off the async executor |

## Exercise

Write an async function `download(id: u32, ms: u64) -> String` that sleeps
for `ms` milliseconds (via `tokio::time::sleep`) and returns a string like
`"file {id} ready"`. In `main`, use `tokio::join!` to run three downloads
with different delays (e.g. 30ms, 10ms, 20ms) concurrently and print all
three results together with the total elapsed time (it should be close to
the *longest* individual delay, not the sum). Then wrap one of the downloads
in a `tokio::select!` against a `tokio::time::sleep` timeout shorter than
its delay, and print which branch won — confirm the download's "ready"
message never prints when the timeout wins.

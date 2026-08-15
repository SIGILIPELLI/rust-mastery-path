# 04 · Production Services

A service that only handles happy-path requests isn't production-ready.
This module adds the three things an orchestrator (Kubernetes, a load
balancer, an on-call human) actually depends on: structured logging via
`tracing`, separate liveness/readiness health checks, and graceful shutdown
that stops accepting new work without dropping in-flight requests.

## Setup

```bash
cargo add axum
cargo add tokio --features full
cargo add tracing tracing-subscriber --features "tracing-subscriber/env-filter"
cargo add serde --features derive
```

## Structured logging with `tracing`

```rust
use tracing::{info, warn};

tracing_subscriber::fmt()
    .with_target(false)
    .init();

info!("listening on {}", addr);
warn!("readiness check failed: warming up");
```

`tracing`'s macros look like `println!` but produce structured events with a
level, a timestamp, and (when you use its span APIs) request-scoped context
— the difference from `println!` matters once logs are aggregated
centrally: `println!` output is just text, `tracing` events can carry
key-value fields a log pipeline can filter and query on.

## Liveness vs. readiness — two different questions

```rust
async fn liveness() -> &'static str {
    "ok"
}

async fn readiness(State(state): State<AppState>) -> Result<Json<Health>, StatusCode> {
    if !state.ready.load(Ordering::Relaxed) {
        warn!("readiness check failed: warming up");
        return Err(StatusCode::SERVICE_UNAVAILABLE);
    }
    let count = state.request_count.fetch_add(1, Ordering::Relaxed);
    Ok(Json(Health { status: "ready", requests_served: count }))
}
```

`/healthz` answers "is the process alive at all" — if this fails, the
orchestrator should kill and restart the container. `/readyz` answers "can
this instance take traffic right now" — if this fails, the instance should
be pulled from the load balancer's rotation but *not* restarted, because
it's likely still starting up (warming a cache, running a migration,
opening a connection pool) and killing it would just repeat the delay.
Collapsing both into one endpoint is a common mistake: it makes the
orchestrator restart instances that were merely still booting.

## Simulated warm-up and the state it flips

```rust
let ready_flag = state.ready.clone();
tokio::spawn(async move {
    tokio::time::sleep(Duration::from_millis(200)).await;
    ready_flag.store(true, Ordering::Relaxed);
    info!("warm-up complete, marking ready");
});
```

`AtomicBool` here stands in for "some real startup cost" — loading a
config, running a migration, warming a cache. The server starts accepting
connections immediately (the socket is bound and listening), but
`/readyz` reports `503` until the flag flips, exactly the signal a load
balancer needs to hold off routing real traffic to this instance.

## Running it

```text
$ cargo run
2026-08-15T07:50:15.426945Z  INFO listening on 127.0.0.1:3003
```

```text
$ curl -s -o /dev/null -w "%{http_code}\n" localhost:3003/readyz
2026-08-15T07:50:15.496552Z  WARN readiness check failed: warming up
503

$ curl -s localhost:3003/readyz
{"status":"ready","requests_served":0}
```

The `WARN` line is printed by the server process itself (to stderr) the
moment the early request comes in — that's the log line an on-call engineer
would see in an aggregator during a rollout, correlating "still 503s" with
"still warming up," instead of guessing.

## Graceful shutdown

```rust
async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    tokio::select! {
        _ = ctrl_c => {
            info!("shutdown signal received");
        }
    }
}

axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();
```

```text
$ kill -INT <pid>
2026-08-15T07:50:15.921770Z  INFO shutdown signal received
```

`with_graceful_shutdown` doesn't kill in-flight requests the instant the
signal arrives — it stops accepting *new* connections and waits for
existing ones to finish before the `axum::serve(...).await` call returns.
This is the mechanism that turns a rolling deploy or pod eviction from
"some requests get an abrupt connection reset" into "requests in flight
complete normally, new ones go to a different instance."

## Rust-specific traps

**`AtomicBool`/`AtomicU64` orderings matter, even when they look
interchangeable.** `Ordering::Relaxed` is used above because there's no
other memory being synchronized alongside the flag — but the moment a
readiness flag is meant to guarantee "and everything it depends on is also
visible" (e.g., a cache that was populated before the flag flipped), you
need `Ordering::Release` on the write and `Ordering::Acquire` on the read,
or a data race is possible even though both operations are individually
atomic.

**`tracing_subscriber::fmt().init()` panics if called twice.** It installs a
global default subscriber; calling `.init()` a second time (common in tests
that each set up their own logging) panics with "a global default trace
dispatcher has already been set." Test code typically uses
`tracing_subscriber::fmt().try_init()` or a `once_cell`/`std::sync::Once`
guard instead.

**Health check handlers must not do expensive work.** A `/readyz` that
actually re-runs a DB query on every check (rather than reading a
pre-computed flag, as above) adds load precisely when the system is already
stressed — orchestrators poll these endpoints every few seconds, so an
expensive check compounds under the exact conditions it's meant to protect
against.

**Signal handling is platform-specific beyond `ctrl_c`.** `tokio::signal::unix::signal(SignalKind::terminate())`
is needed to also catch `SIGTERM` (what Kubernetes actually sends on pod
eviction, not `SIGINT`) — `signal::ctrl_c()` alone only catches Ctrl+C /
`SIGINT`, which is fine for local dev but incomplete for a real deployment.

## Cheat sheet

| Concern | Mechanism |
|---|---|
| Structured logs | `tracing` + `tracing_subscriber::fmt().init()` |
| "Is the process alive" | `/healthz` — cheap, always `200` unless truly wedged |
| "Can this instance serve traffic" | `/readyz` — reflects real startup/dependency state |
| Stop new work, finish in-flight | `axum::serve(...).with_graceful_shutdown(fut)` |
| Catch `SIGTERM` (not just Ctrl+C) | `tokio::signal::unix::signal(SignalKind::terminate())` |
| Thread-safe counters without locks | `std::sync::atomic::{AtomicU64, AtomicBool}` |

## Exercise

Add a `SIGTERM` handler alongside the existing `ctrl_c` branch in
`shutdown_signal`, using `tokio::select!` to race both, so the service
shuts down gracefully on either signal. Test it by starting the server,
running `kill -TERM <pid>` in another terminal, and confirming the same
"shutdown signal received" log line appears.

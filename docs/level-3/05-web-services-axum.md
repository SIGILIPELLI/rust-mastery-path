# 05 · Web Services with axum

`axum` is the web framework built by the Tokio team on top of `tower`'s
service/middleware model. It leans hard on Rust's type system: routes are
async functions, and the parameters those functions ask for — a path
segment, a JSON body, a piece of shared state — are declared as typed
"extractors" that axum wires up for you. Get the types right and the
framework does the boilerplate; get them wrong and it's a compile error, not
a 500 at 3am.

## Setup

```bash
cargo new tasks-api
cd tasks-api
cargo add axum tokio --features tokio/full
cargo add serde --features derive
```

## A small in-memory task API

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};

#[derive(Clone, Serialize, Deserialize, Debug)]
struct Task {
    id: u32,
    title: String,
}

#[derive(Deserialize)]
struct NewTask {
    title: String,
}

#[derive(Clone)]
struct AppState {
    tasks: Arc<Mutex<Vec<Task>>>,
}

async fn list_tasks(State(state): State<AppState>) -> Json<Vec<Task>> {
    let tasks = state.tasks.lock().unwrap();
    Json(tasks.clone())
}

async fn create_task(
    State(state): State<AppState>,
    Json(payload): Json<NewTask>,
) -> (StatusCode, Json<Task>) {
    let mut tasks = state.tasks.lock().unwrap();
    let id = tasks.len() as u32 + 1;
    let task = Task { id, title: payload.title };
    tasks.push(task.clone());
    (StatusCode::CREATED, Json(task))
}

async fn get_task(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> Result<Json<Task>, StatusCode> {
    let tasks = state.tasks.lock().unwrap();
    tasks
        .iter()
        .find(|t| t.id == id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

#[tokio::main]
async fn main() {
    let state = AppState { tasks: Arc::new(Mutex::new(Vec::new())) };

    let app = Router::new()
        .route("/tasks", get(list_tasks).post(create_task))
        .route("/tasks/{id}", get(get_task))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

Every handler's parameter list *is* its contract: `State(state): State<AppState>`
pulls the shared state out, `Json(payload): Json<NewTask>` deserializes the
request body and rejects the request with `400 Bad Request` automatically if
it doesn't match, `Path(id): Path<u32>` parses the URL segment and rejects
with `400` if it isn't a valid `u32`. None of that validation code is
visible in the handler body — axum generates it from the types.

## Running it and hitting it with curl

```text
$ cargo run
listening on 127.0.0.1:3001
```

```text
$ curl -s -X POST localhost:3001/tasks -H 'content-type: application/json' -d '{"title":"write docs"}'
{"id":1,"title":"write docs"}

$ curl -s -X POST localhost:3001/tasks -H 'content-type: application/json' -d '{"title":"ship release"}'
{"id":2,"title":"ship release"}

$ curl -s localhost:3001/tasks
[{"id":1,"title":"write docs"},{"id":2,"title":"ship release"}]

$ curl -s localhost:3001/tasks/1
{"id":1,"title":"write docs"}

$ curl -s -o /dev/null -w "%{http_code}\n" localhost:3001/tasks/99
404
```

The `404` for a missing task comes from `get_task` returning
`Err(StatusCode::NOT_FOUND)` — axum's `IntoResponse` impl for `StatusCode`
turns that directly into an empty response with that status, no manual
response-building required.

## Rust-specific traps

**`Mutex` across `.await` points.** `state.tasks.lock().unwrap()` returns a
`std::sync::MutexGuard`, which is not `Send`. Hold one across an `.await` in
an async handler and the compiler rejects the whole future as non-`Send` —
confusing because the error surfaces far from the `lock()` call, often at
the router's `.route()` line. The fix here is deliberate: every lock is
acquired, used, and dropped before any `.await`, never held across one. For
real workloads needing to hold a lock across awaits, use `tokio::sync::Mutex`
instead.

**Extractor order matters.** Axum requires the body-consuming extractor
(`Json<T>`, `Bytes`, etc.) to be the *last* argument in a handler's
signature — it consumes the request, so anything after it can't extract
from what's left. Put `Json(payload)` before `State(state)` and you get a
trait-bound error that doesn't obviously point at ordering.

**Route paths changed syntax.** Axum 0.7+ uses `/tasks/{id}` for path
parameters, not the older `/tasks/:id`. Copying examples from slightly older
tutorials produces a router that compiles but 404s on every request to that
route, because the whole path is matched literally.

**Cloning `Arc<Mutex<T>>` state per request.** `AppState` derives `Clone`,
and axum clones it for every handler invocation. That's cheap here — it's
only cloning an `Arc` pointer, not the `Vec<Task>` behind it — but it's easy
to accidentally wrap the wrong field in a way that makes each clone deep.
Keep anything expensive behind `Arc`.

## Cheat sheet

| Piece | Role |
|---|---|
| `Router::new().route(path, method(handler))` | Registers a handler for a path + HTTP method |
| `State<T>` | Extracts shared app state, requires `.with_state(t)` |
| `Path<T>` | Extracts and parses URL path segments |
| `Json<T>` | Extracts (request) or serializes (response) a JSON body |
| `impl IntoResponse` | What a handler may return: tuples, `Json`, `StatusCode`, `Result<T, E>` |
| `axum::serve(listener, app)` | Runs the server on a bound `TcpListener` |

## Exercise

Add a `DELETE /tasks/{id}` route backed by a `delete_task` handler that
removes the matching task from the `Vec` and returns `204 No Content`, or
`404` if the id isn't found. Test it with
`curl -i -X DELETE localhost:3001/tasks/1` followed by
`curl -s localhost:3001/tasks` to confirm the task is gone.

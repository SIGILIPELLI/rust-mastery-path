# 01 · Concurrency

Rust's headline safety pitch isn't just "no null pointer dereferences" — it's
"no data races, enforced at compile time." Most languages let you spawn
threads freely but leave shared-mutable-state bugs to runtime (or to a
separate tool like a race detector) to catch. Rust's ownership and borrowing
rules from [Level 2](../level-2/01-ownership-borrowing.md) extend directly
into concurrent code: the same rules that stop you from having two mutable
references to a value in a single thread also stop you from handing two
threads a mutable reference to the same value without synchronization. This
module covers `std::thread`, message passing with channels, and shared state
with `Arc<Mutex<T>>`.

## Spawning a thread

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..=3 {
            println!("spawned thread: {i}");
        }
    });

    for i in 1..=2 {
        println!("main thread: {i}");
    }

    handle.join().unwrap(); // block until the spawned thread finishes
    println!("done");
}
```

```text
main thread: 1
main thread: 2
spawned thread: 1
spawned thread: 2
spawned thread: 3
done
```

`thread::spawn` returns a `JoinHandle` immediately — the spawned thread runs
concurrently with `main`, which is *why* the interleaving of the two
`println!` sequences isn't guaranteed (you might see different ordering on a
different run). `.join()` blocks the calling thread until the spawned one
finishes; without it, `main` could exit before the spawned thread ever gets
scheduled, silently dropping its output. `.unwrap()` is needed because
`.join()` returns a `Result` — it's `Err` if the spawned thread panicked.

## The trap: closures and `'static`

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("{:?}", data);
    });

    handle.join().unwrap();
}
```

```text
error[E0373]: closure may outlive the current function, but it borrows `data`, which is owned by the current function
 --> src/main.rs:6:32
  |
6 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `data`
7 |         println!("{:?}", data);
  |                          ---- `data` is borrowed here
  |
help: to force the closure to take ownership of `data` (and any other referenced variables), use the `move` keyword
```

`thread::spawn` requires its closure to be `'static` — it might run for
longer than the current stack frame, so it can't hold a borrow of anything
that could be dropped before it finishes. The compiler's own suggestion is
the fix:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("{:?}", data); // `data` is now owned by the closure
    });

    handle.join().unwrap();
}
```

```text
[1, 2, 3]
```

`move` transfers ownership of `data` into the closure instead of borrowing
it — after this line, `main` can no longer use `data` itself, which is
exactly the ownership rule you'd expect: only one owner, and it moved.

## Message passing with channels

Rust's standard library favors "share memory by communicating" — send
owned values between threads over a channel rather than reaching into
shared memory directly.

```rust
use std::sync::mpsc; // multi-producer, single-consumer
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    for id in 0..3 {
        let tx = tx.clone(); // each worker gets its own sender handle
        thread::spawn(move || {
            thread::sleep(Duration::from_millis(10 * id));
            tx.send(format!("worker {id} done")).unwrap();
        });
    }
    drop(tx); // drop the original sender -- see below

    for msg in rx {
        println!("{msg}");
    }
}
```

```text
worker 0 done
worker 1 done
worker 2 done
```

Two details that trip people up the first time:

- `tx.clone()` makes a new *handle* to the same channel, not a new channel —
  `mpsc` allows multiple producers, and every clone can send into the same
  receiver.
- `for msg in rx` iterates until every sender is dropped. If you forget the
  `drop(tx)` line, the *original* `tx` is still alive (it's never moved
  anywhere), so the receiver's iterator blocks forever waiting for a message
  that will never come — the program hangs instead of finishing. This is a
  real, easy-to-hit bug: always make sure every clone of a sender eventually
  goes out of scope or gets dropped.

## Shared state with `Arc<Mutex<T>>`

Channels move ownership; sometimes you genuinely need several threads to
read and write the *same* piece of memory — a shared counter, a cache, a
connection pool. `Mutex<T>` (mutual exclusion) allows only one thread to
access the data at a time, and `Arc<T>` (atomic reference counted — the
thread-safe sibling of [Level 2's `Rc<T>`](../level-2/07-smart-pointers.md))
lets multiple threads jointly own the `Mutex`.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap(); // blocks until the lock is free
            *num += 1;
        }); // lock is released here, when `num` goes out of scope
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("total: {}", *counter.lock().unwrap());
}
```

```text
total: 10
```

`counter.lock()` returns a `Result<MutexGuard<T>, _>` — it's `Err` only if
the mutex was "poisoned" (a previous thread panicked while holding the
lock), which `.unwrap()` treats as fatal here. `MutexGuard` derefs to the
inner value and automatically releases the lock via `Drop` when it goes out
of scope — you never call an explicit `unlock()`, which also means the lock
is released even if the thread panics partway through (though the mutex
becomes poisoned in that case).

Why does `Arc::clone(&counter)` compile inside a `move` closure when a plain
reference wouldn't? Each iteration creates a *new* `Arc` handle (an
increment of the reference count) and moves *that* new handle into the
closure — the original `counter` binding in `main` is untouched, so the next
loop iteration can clone it again. This is the same shadowing pattern used
in [Level 2's `Rc` module](../level-2/07-smart-pointers.md), just with the
thread-safe type.

## `Mutex<T>` vs a plain shared reference — why the compiler forces this

```rust
use std::thread;

fn main() {
    let mut counter = 0;
    let mut handles = vec![];

    for _ in 0..10 {
        handles.push(thread::spawn(|| {
            counter += 1; // ERROR: `counter` is captured by reference, and
                          // multiple threads cannot each get a `&mut` to it
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

This fails to compile with the same category of error as the `'static` trap
above — `counter` isn't safe to mutate from multiple closures without
synchronization, and the compiler catches this *before* the program ever
runs, unlike in languages where this exact bug becomes a runtime data race
that only shows up under load. `Mutex` is how you tell the compiler "yes, I
know several threads touch this — here's the lock that makes it safe."

## Deadlocks: the trap `Mutex` doesn't protect against

`Mutex` prevents data races, but it cannot prevent deadlocks — two threads
each holding a lock the other one needs:

```text
Thread A: locks `first`, then tries to lock `second`
Thread B: locks `second`, then tries to lock `first`

If both grab their first lock before either reaches the second,
neither can ever proceed -- this compiles fine and simply hangs forever.
```

The practical rule: always acquire multiple locks in the same, consistent
order across every thread that takes more than one, or restructure the code
so no thread ever needs to hold two locks at once.

## Cheat sheet

| Tool | Use when |
|------|----------|
| `thread::spawn` | Run a closure concurrently; returns a `JoinHandle` |
| `handle.join()` | Block until a spawned thread finishes; propagates panics as `Err` |
| `move \|\| { .. }` | Force a closure to take ownership instead of borrowing (required for `'static`) |
| `mpsc::channel()` | Move owned values between threads without sharing memory |
| `tx.clone()` | Give another thread/producer its own sender handle |
| `Arc<T>` | Thread-safe shared ownership (like `Rc`, but atomic) |
| `Mutex<T>` | Runtime-enforced exclusive access to a value |
| `counter.lock().unwrap()` | Block until the lock is free; returns a `MutexGuard` |

## Exercise

Write a program that spawns 5 threads, each simulating a "download" by
sleeping a duration (use `thread::sleep` with each thread's index times
20ms) and then sending a `String` message like `"file 3 downloaded"` over an
`mpsc::channel`. In `main`, collect and print every message as it arrives,
then print a final summary line with the total count. Then rewrite it to
instead use `Arc<Mutex<Vec<String>>>` shared between the threads (each
thread pushes its own message directly into the shared `Vec` instead of
sending it over a channel) and confirm both versions produce the same
final count.

---
title: "Rust Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Rust cheat sheet: 33 quick-reference tables from every level of the Rust Mastery Path, each linked to its full lesson."
---

# Rust Cheat Sheet

Quick-reference tables from **33 lessons** of the free Rust Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Variables, Types &amp; Ownership Basics](level-1/02-variables-types-ownership.md)

| Type | Category | Example | Copy? |
|------|----------|---------|-------|
| `i32`, `u32`, `i64`, ... | Signed/unsigned integer | `let x: i32 = -7;` | Yes |
| `f64`, `f32` | Floating point | `let pi: f64 = 3.14;` | Yes |
| `bool` | Boolean | `let ok = true;` | Yes |
| `char` | Single Unicode scalar | `let c = 'R';` | Yes |
| `(T1, T2, ...)` | Tuple | `let t = (1, "a", true);` | Yes, if all elements are |
| `[T; N]` | Fixed-size array | `let a = [1, 2, 3];` | Yes, if `T` is |
| `String` | Growable, heap-allocated text | `String::from("hi")` | No |
| `Vec<T>` | Growable, heap-allocated list | `vec![1, 2, 3]` | No |

### [03 · Control Flow](level-1/03-control-flow.md)

| Construct | Produces a value? | Typical use |
|-----------|--------------------|-------------|
| `if`/`else if`/`else` | Yes, can be used in `let` | Branching, conditional value |
| `loop` | Yes, via `break value` | Repeat until a condition inside is met |
| `while` | No | Repeat while a condition holds |
| `for x in 0..n` | No | Repeat a known number of times |
| `for x in collection.iter()` | No | Visit every element of a collection |
| `continue` | — | Skip to the next loop iteration |
| `break` | Optional value (in `loop`) | Exit a loop early |

### [04 · Functions](level-1/04-functions.md)

| Concept | Example | Notes |
|---------|---------|-------|
| Declaration | `fn add(a: i32, b: i32) -> i32 { a + b }` | Parameters and return type always explicit |
| Implicit return | `a + b` (no `;`) | Last expression in the body is the return value |
| Explicit return | `return a + b;` | Needed for early exits, requires a semicolon |
| No return value | `fn log(msg: &str) { ... }` | Implicitly returns `()`, the unit type |
| Block as expression | `let y = { ...; last_expr };` | A `{ }` block evaluates to its last unterminated expression |

### [05 · Structs &amp; Enums](level-1/05-structs-enums.md)

| Concept | Syntax |
|---------|--------|
| Define a struct | `struct Name { field: Type, ... }` |
| Instantiate | `Name { field: value, ... }` |
| Method | `impl Name { fn method(&self) -> T { ... } }` |
| Associated function | `impl Name { fn new(...) -> Name { ... } }` |
| Define an enum | `enum Name { VariantA, VariantB(Type), ... }` |
| Match on an enum | `match value { Name::VariantA => ..., ... }` |
| Auto-derive traits | `#[derive(Debug, Clone, PartialEq)]` |

### [06 · Collections (Vec, String, HashMap)](level-1/06-collections.md)

| Type | Owns its data? | Grows? | Common methods |
|------|-----------------|--------|-----------------|
| `Vec<T>` | Yes | Yes | `.push()`, `.pop()`, `.get()`, `.len()`, `.iter()` |
| `String` | Yes | Yes | `.push_str()`, `.push()`, `+`, `format!`, `.chars()` |
| `&str` | No (borrowed) | No | `.len()`, `.chars()`, slicing (`&s[0..3]`) |
| `HashMap<K, V>` | Yes | Yes | `.insert()`, `.get()`, `.entry().or_insert()`, `.iter()` |

### [07 · Error Handling Basics (Option, Result)](level-1/07-error-handling-basics.md)

| Method | Works on | Behavior |
|--------|----------|----------|
| `.is_some()` / `.is_none()` | `Option` | Check without consuming |
| `.is_ok()` / `.is_err()` | `Result` | Check without consuming |
| `.unwrap()` | Both | Returns value, **panics** on `None`/`Err` |
| `.expect("msg")` | Both | Like `.unwrap()`, with a custom panic message |
| `.unwrap_or(default)` | Both | Returns value, or `default` on `None`/`Err` |
| `.unwrap_or_else(\|\| ...)` | Both | Returns value, or computes a fallback lazily |
| `if let Some(x) = opt` | `Option` | Handle just the "present" case |
| `if let Ok(x) = res` | `Result` | Handle just the "success" case |
| `?` | Both | Propagate `None`/`Err` out of the current function |

### [08 · Pattern Matching](level-1/08-pattern-matching.md)

| Construct | When to use |
|-----------|-------------|
| `match` | Exhaustively handle every possible variant/value |
| `\|` (OR pattern) | Match several literal values in one arm |
| `a..=b` | Match an inclusive numeric range |
| Match guard (`if cond`) | Add an extra condition to a pattern |
| `if let` | Handle just one pattern, ignore the rest |
| `while let` | Loop for as long as a pattern keeps matching |
| `..` in a struct pattern | Ignore the remaining fields |

### [09 · Modules &amp; Cargo Project Structure](level-1/09-modules-cargo.md)

| Task | Command/Syntax |
|------|-----------------|
| New project | `cargo new project_name` |
| Build | `cargo build` |
| Build and run | `cargo run` |
| Run tests | `cargo test` |
| Define a module | `mod name { ... }` or a separate `name.rs` file |
| Expose an item | `pub fn ...` / `pub struct ...` |
| Bring a path into scope | `use path::to::item;` |
| Add a dependency | Edit `[dependencies]` in `Cargo.toml` |

## Level 2 · Intermediate

### [01 · Ownership &amp; Borrowing Deep Dive](level-2/01-ownership-borrowing.md)

| Situation | Allowed? |
|-----------|----------|
| Multiple `&T` to the same value | Yes |
| One `&mut T` to the same value | Yes |
| `&T` and `&mut T` to the same value at once | No |
| Two `&mut T` to the same value at once | No |
| `&mut` to two different fields of the same struct | Yes |
| Returning a reference to a local variable | No (dangling) — return owned data instead |
| Reference outliving the data it points to | No — caught at compile time |

### [02 · Lifetimes](level-2/02-lifetimes.md)

| Syntax | Meaning |
|--------|---------|
| `&'a str` | A reference valid for lifetime `'a` |
| `fn f<'a>(x: &'a str) -> &'a str` | Output lifetime tied to input `x` |
| `struct S<'a> { field: &'a str }` | Struct can't outlive the data `field` borrows |
| `'static` | Valid for the whole program (string literals, leaked/global data) |
| Elision rule 1 | Each elided input reference gets its own lifetime |
| Elision rule 2 | One input lifetime → applied to all elided outputs |
| Elision rule 3 | `&self`/`&mut self` lifetime → applied to all elided outputs |

### [04 · Closures &amp; Iterators](level-2/04-closures-iterators.md)

| Method | Purpose | Consumes the iterator? |
|--------|---------|--------------------------|
| `.map(f)` | Transform each item | No — lazy adapter |
| `.filter(pred)` | Keep matching items | No — lazy adapter |
| `.enumerate()` | Pair items with their index | No — lazy adapter |
| `.zip(other)` | Pair items from two iterators | No — lazy adapter |
| `.take(n)` / `.skip(n)` | Limit / skip items | No — lazy adapter |
| `.collect()` | Build a `Vec`/`String`/etc. from the iterator | Yes |
| `.sum()` / `.fold()` | Reduce to a single value | Yes |
| `.any(pred)` / `.all(pred)` | Boolean check, short-circuits | Yes |
| `.for_each(f)` | Run a closure per item, no return value | Yes |

### [05 · Error Handling Advanced](level-2/05-error-handling-advanced.md)

| Tool | Use when |
|------|----------|
| `enum AppError { ... }` + `impl From<X> for AppError` | Multiple error sources, want to `match` on which one |
| `impl std::error::Error for E` | Making your error type interoperate with the ecosystem |
| `Box<dyn Error>` | Application code, don't need to distinguish error variants |
| `.map_err(\|e\| ...)` | One-off conversion or adding context at a single call site |
| `?` | Propagate any error convertible via `From` into the function's return type |
| `panic!` / `.expect()` | Unrecoverable bugs, not user-facing failures |

### [06 · Testing in Rust](level-2/06-testing.md)

| Macro/attribute | Purpose |
|-----------------|---------|
| `#[test]` | Marks a function as a test case |
| `#[cfg(test)]` | Compiles a module only when testing |
| `assert!(cond)` | Fails if `cond` is false |
| `assert_eq!(a, b)` / `assert_ne!(a, b)` | Fails if not equal / equal, prints both values |
| `#[should_panic]` | Test passes only if the function panics |
| `#[ignore]` | Skips the test unless `--ignored` is passed |
| `cargo test <name>` | Run tests whose name contains `<name>` |
| `cargo test -- --nocapture` | Show `println!` output from passing tests |

### [08 · Files &amp; the Standard Library](level-2/08-files-stdlib.md)

| Task | Tool |
|------|------|
| Read a whole file into a `String` | `fs::read_to_string(path)` |
| Write a whole file at once | `fs::write(path, contents)` |
| Read line-by-line, efficiently | `BufReader::new(file).lines()` |
| Write many lines, efficiently | `BufWriter::new(file)` + `writeln!()` |
| Create-or-truncate, append instead | `OpenOptions::new().append(true)...` |
| Check *why* an I/O op failed | `err.kind() == ErrorKind::NotFound` etc. |
| Build a path portably | `PathBuf::from(...).push(...)` / `.join(...)` |
| Read one line from the user | `io::stdin().read_line(&mut buf)` |

### [09 · Crates &amp; Dependency Management](level-2/09-crates-dependencies.md)

| Task | Command / syntax |
|------|-------------------|
| Add a dependency | `cargo add <crate>` or edit `[dependencies]` directly |
| Pin an exact version | `"=1.2.3"` |
| Allow compatible updates (default) | `"1.2.3"` (same as `"^1.2.3"`) |
| Update within allowed ranges | `cargo update` |
| Test-only dependency | `[dev-dependencies]` |
| Opt into optional functionality | `features = ["derive"]` |
| Reference a local crate | `{ path = "../other_crate" }` |
| Group multiple crates | `[workspace]` + `members = [...]` in the root `Cargo.toml` |

## Level 3 · Advanced

### [01 · Concurrency](level-3/01-concurrency.md)

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

### [02 · Async/Await Basics](level-3/02-async-await-basics.md)

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

### [03 · Building CLIs with clap](level-3/03-clap-cli.md)

| Attribute | Effect |
|---|---|
| `#[derive(Parser)]` | Generates `parse()` / `try_parse()` on the struct |
| `#[command(subcommand)]` | Field's enum becomes the subcommand dispatch |
| `#[arg(short, long)]` | Enables both `-x` and `--xyz` forms |
| `#[arg(default_value_t = v)]` | Optional argument with a typed default |
| `Option<T>` field | Argument is optional, `None` if absent |
| `Vec<T>` field | Argument may be repeated / collects multiple values |
| `#[command(version, about)]` | Pulls `--version`/`--help` text from `Cargo.toml` / doc comments |

### [04 · Databases with rusqlite](level-3/04-databases-rusqlite.md)

| Call | Purpose |
|---|---|
| `Connection::open("file.db")` | Open (creating if absent) a database file |
| `Connection::open_in_memory()` | Ephemeral `:memory:` database, useful for tests |
| `conn.execute(sql, params)` | Run INSERT/UPDATE/DELETE/DDL, returns rows affected |
| `conn.prepare(sql)` | Compile a statement for reuse or row iteration |
| `stmt.query_map(params, closure)` | Row iterator, closure maps each row to a value |
| `row.get::<_, T>(i)` | Typed column extraction by index, fallible |
| `params![a, b, ...]` | Macro for building a heterogeneous parameter list |

### [05 · Web Services with axum](level-3/05-web-services-axum.md)

| Piece | Role |
|---|---|
| `Router::new().route(path, method(handler))` | Registers a handler for a path + HTTP method |
| `State<T>` | Extracts shared app state, requires `.with_state(t)` |
| `Path<T>` | Extracts and parses URL path segments |
| `Json<T>` | Extracts (request) or serializes (response) a JSON body |
| `impl IntoResponse` | What a handler may return: tuples, `Json`, `StatusCode`, `Result<T, E>` |
| `axum::serve(listener, app)` | Runs the server on a bound `TcpListener` |

### [06 · Unsafe Rust](level-3/06-unsafe-rust.md)

| Operation | Why it needs `unsafe` |
|---|---|
| `*raw_ptr` | Compiler can't verify the pointer is valid and non-dangling |
| `unsafe fn foo()` call | Function has documented preconditions the compiler can't check |
| `static mut X` access | Compiler can't prove no data race across threads |
| `unsafe trait` impl | Implementer is asserting an invariant the trait promises |
| union field read | Compiler can't verify which variant is currently active |
| `std::ptr::addr_of!(X)` | Safe way to get a raw pointer to a static without a reference |

### [07 · Design Patterns in Rust](level-3/07-design-patterns.md)

| Pattern | Rust mechanism |
|---|---|
| Builder | Chained `self -> Self` methods, final `build(self) -> T` |
| Strategy | `&dyn Trait` or generic `impl Trait` parameter |
| Newtype | Tuple struct wrapping one field, own trait impls |
| Typestate | Generic struct parameterized by marker types + `PhantomData` |
| Singleton (rare in Rust) | `once_cell`/`std::sync::OnceLock`, not a class-level static |
| Observer | Channel (`mpsc`) or callback `Vec<Box<dyn Fn(...)>>` |

### [08 · Testing &amp; Benchmarking](level-3/08-testing-benchmarking.md)

| Command | Runs |
|---|---|
| `cargo test` | `#[test]` fns in `src/`, plus `tests/*.rs`, plus doc tests |
| `cargo test foo` | Only tests whose name contains `foo` |
| `cargo test -- --nocapture` | Also shows `println!` output from passing tests |
| `cargo bench` | `benches/*.rs`, using the harness declared in `Cargo.toml` |
| `#[should_panic(expected = "...")]` | Test passes only if the panic message contains the substring |
| `#[ignore]` | Skip by default; run explicitly with `cargo test -- --ignored` |

### [09 · Generics &amp; Trait Objects](level-3/09-generics-trait-objects.md)

| | Generics (`<T: Trait>`) | Trait objects (`dyn Trait`) |
|---|---|---|
| Dispatch | Static, resolved at compile time | Dynamic, vtable lookup at runtime |
| Binary size | One copy per concrete `T` | One shared implementation |
| Collections | `Vec<T>` — one concrete type | `Vec<Box<dyn Trait>>` — mixed types |
| Requires | Trait bound compiles per call site | Trait must be object-safe |
| Typical pointer | `&T`, owned `T` | `&dyn`, `Box<dyn>`, `Rc<dyn>`/`Arc<dyn>` |

## Level 4 · Master

### [01 · Advanced Async Patterns](level-4/01-advanced-async-patterns.md)

| Primitive | Behavior |
|---|---|
| `tokio::join!(a, b, c)` | Runs all concurrently, waits for all, keeps every result |
| `tokio::try_join!(a, b, c)` | Like `join!`, but short-circuits and cancels the rest on first `Err` |
| `tokio::select! { .. }` | Races branches, takes first ready, drops the others |
| `handle.abort()` | Requests cancellation at the task's next `.await` point |
| `tokio::time::timeout(d, fut)` | Races `fut` against a timer, `Err` on timeout |
| `JoinError::is_cancelled()` | True if a joined task was aborted rather than panicked |

### [02 · Performance Optimization](level-4/02-performance-optimization.md)

| Technique | When it helps |
|---|---|
| `String::with_capacity(n)` | Building a string in a loop with a known/estimable final size |
| Generic `<F: Fn(...)>` over `Box<dyn Fn(...)>` | Hot loops calling the same closure repeatedly |
| `--release` builds | Any performance comparison, always |
| `cargo bench` / criterion | Precise, statistically sound measurement (Level 3, module 08) |
| `Vec::with_capacity(n)` | Same idea as `String::with_capacity`, for growable collections |
| Avoid `.clone()` in loops | Each clone is a real allocation + copy; borrow instead where possible |

### [03 · Macros](level-4/03-macros.md)

| | `macro_rules!` | proc macro |
|---|---|---|
| Input | Token patterns you define | Full parsed AST (`syn::DeriveInput` etc.) |
| Crate requirement | None — lives in any crate | Separate crate with `proc-macro = true` |
| Typical use | Repetitive syntax, small DSLs | `#[derive(...)]`, custom attributes |
| Hygiene | Automatic (caller/macro names don't collide) | Manual — you construct identifiers yourself |
| Debugging | `cargo expand`, careful pattern reading | `cargo expand`, `eprintln!` inside the macro fn |

### [04 · Production Services](level-4/04-production-services.md)

| Concern | Mechanism |
|---|---|
| Structured logs | `tracing` + `tracing_subscriber::fmt().init()` |
| "Is the process alive" | `/healthz` — cheap, always `200` unless truly wedged |
| "Can this instance serve traffic" | `/readyz` — reflects real startup/dependency state |
| Stop new work, finish in-flight | `axum::serve(...).with_graceful_shutdown(fut)` |
| Catch `SIGTERM` (not just Ctrl+C) | `tokio::signal::unix::signal(SignalKind::terminate())` |
| Thread-safe counters without locks | `std::sync::atomic::{AtomicU64, AtomicBool}` |

### [05 · Security &amp; Memory Safety](level-4/05-security-memory-safety.md)

| Risk | Rust tool |
|---|---|
| Silent integer overflow in release | `checked_add`/`checked_sub`/`checked_mul`, or `saturating_*` |
| Untrusted input crashing the process | `Result`-returning parse + explicit range checks, never `unwrap()` |
| Secrets lingering in freed memory | `zeroize::Zeroize`, or a custom `Drop` calling it |
| Use-after-free, double-free | Prevented by ownership — not something to defend against manually |
| Data races on shared mutable state | Prevented by `Send`/`Sync` + the borrow checker |

### [06 · Testing at Scale &amp; CI](level-4/06-testing-at-scale-ci.md)

| Tool | Command | Catches |
|---|---|---|
| `cargo test` | Unit, integration, and doc tests | Logic bugs, regressions |
| `cargo clippy -- -D warnings` | Idiom / correctness lints | Non-idiomatic or subtly wasteful code |
| `cargo fmt --check` | Formatting drift | Inconsistent style, noisy diffs |
| `cargo test --doc` | Just doc-comment examples | Stale examples in `///` comments |
| `dtolnay/rust-toolchain@stable` | GitHub Actions setup step | Reproducible CI toolchain |

### [07 · Deployment with Docker](level-4/07-deployment-docker.md)

| Concern | Fix |
|---|---|
| Slow rebuilds on every source change | Copy `Cargo.toml`/`Cargo.lock` and build deps before copying `src/` |
| Large final image | Multi-stage build, copy only the compiled binary |
| Binary size | `strip = true`, `lto = true` in `[profile.release]` |
| `FROM scratch` fails at runtime | Build against `x86_64-unknown-linux-musl` for a static binary |
| Running as root | `useradd` + `USER` directive in the Dockerfile |
| Runtime vs. compile-time config | `std::env::var` (runtime) vs. `env!()` macro (compile-time, baked in) |

### [08 · FFI Basics](level-4/08-ffi-basics.md)

| Direction | Mechanism |
|---|---|
| Call C from Rust | `unsafe extern "C" { fn foo(...); }`, call inside `unsafe { }` |
| Expose Rust to C | `#[unsafe(no_mangle)] pub extern "C" fn foo(...)` |
| Share struct layout | `#[repr(C)]` on the struct |
| Pass a string to C | `std::ffi::CString`, not a raw `&str` cast |
| Pass a buffer + length | `*const T` / `*mut T` plus a separate `usize` length |
| Compile a C source file | `cc` crate in `build.rs` |

### [09 · Workspace Organization](level-4/09-workspace-organization.md)

| Command | Effect |
|---|---|
| `cargo build` (from root) | Builds every workspace member |
| `cargo run -p cli` | Runs the `cli` member's binary specifically |
| `cargo test -p core` | Tests only the `core` member |
| `cargo tree -p cli` | Shows `cli`'s full dependency tree, deduped |
| `version.workspace = true` | Inherits `[workspace.package].version` |
| `dep.workspace = true` | Inherits the version/features from `[workspace.dependencies]` |
| `core = { path = "../core" }` | Local sibling-crate dependency, always current |

---

Go deeper with the full [Rust Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).

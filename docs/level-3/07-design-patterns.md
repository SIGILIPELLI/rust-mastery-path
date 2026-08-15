# 07 · Design Patterns in Rust

Classic OOP design patterns exist to work around limitations Rust doesn't
have — inheritance-based polymorphism, null references, mutable shared
state as the default. Rust's own idioms cover the same problems with traits,
`Option`, ownership, and the type system doing compile-time checking that
GoF patterns did at runtime (or not at all). This module walks four patterns
that show up constantly in real Rust code: builder, strategy, newtype, and
typestate.

## Builder: construct in steps, validate at the end

```rust
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    tls: bool,
}

struct ServerBuilder {
    host: String,
    port: u16,
    tls: bool,
}

impl ServerBuilder {
    fn new() -> Self {
        ServerBuilder { host: "127.0.0.1".into(), port: 8080, tls: false }
    }
    fn host(mut self, host: &str) -> Self {
        self.host = host.to_string();
        self
    }
    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }
    fn tls(mut self, tls: bool) -> Self {
        self.tls = tls;
        self
    }
    fn build(self) -> Server {
        Server { host: self.host, port: self.port, tls: self.tls }
    }
}

fn main() {
    let server = ServerBuilder::new().host("0.0.0.0").port(443).tls(true).build();
    println!("{:?}", server);
}
```

```text
Server { host: "0.0.0.0", port: 443, tls: true }
```

Each builder method takes `self` by value and returns `Self`, so calls chain
without a mutable local variable. This is the pattern Rust reaches for
instead of optional constructor arguments — `Server` has no `Default` or
`new()` with five parameters where three are usually the same value.

## Strategy: swap behavior through a trait object

```rust
trait Discount {
    fn apply(&self, cents: u32) -> u32;
}

struct NoDiscount;
impl Discount for NoDiscount {
    fn apply(&self, cents: u32) -> u32 {
        cents
    }
}

struct PercentOff(u32);
impl Discount for PercentOff {
    fn apply(&self, cents: u32) -> u32 {
        cents - (cents * self.0 / 100)
    }
}

fn checkout(cents: u32, strategy: &dyn Discount) -> u32 {
    strategy.apply(cents)
}

fn main() {
    let strategies: Vec<Box<dyn Discount>> = vec![Box::new(NoDiscount), Box::new(PercentOff(20))];
    for s in &strategies {
        println!("{}", checkout(2500, s.as_ref()));
    }
}
```

```text
2500
2000
```

`&dyn Discount` is a trait object — a fat pointer (data pointer + vtable)
that lets `checkout` accept any type implementing `Discount` without knowing
which one at compile time. This is Rust's answer to strategy: runtime
polymorphism through a trait, not a class hierarchy. Prefer `impl Discount`
generics when you know the concrete type at each call site — `dyn` earns its
keep specifically when you need a heterogeneous collection like the `Vec`
above.

## Newtype: a zero-cost wrapper with its own semantics

```rust
struct Cents(u32);
impl std::fmt::Display for Cents {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "${}.{:02}", self.0 / 100, self.0 % 100)
    }
}

fn main() {
    println!("{}", Cents(2599));
}
```

```text
$25.99
```

`Cents(u32)` is a tuple struct with one field — same memory layout as a bare
`u32`, zero runtime overhead, but it's a distinct type. A function taking
`Cents` can't accidentally be called with a raw `u32` meaning "dollars" or
"quantity"; the newtype makes the unit part of the type signature instead
of a comment. It's also the standard way to implement a foreign trait (like
`Display`) for a foreign type (like `u32`) — Rust's orphan rule forbids
`impl Display for u32` directly, but a local newtype sidesteps it.

## Typestate: illegal states unrepresentable at compile time

```rust
struct Draft;
struct Published;

struct Post<State> {
    text: String,
    _state: std::marker::PhantomData<State>,
}

impl Post<Draft> {
    fn new(text: &str) -> Self {
        Post { text: text.to_string(), _state: std::marker::PhantomData }
    }
    fn publish(self) -> Post<Published> {
        Post { text: self.text, _state: std::marker::PhantomData }
    }
}

impl Post<Published> {
    fn text(&self) -> &str {
        &self.text
    }
}

fn main() {
    let draft = Post::<Draft>::new("hello world");
    let published = draft.publish();
    println!("{}", published.text());
}
```

```text
hello world
```

`text()` only exists on `Post<Published>` — there is no way to call it on a
`Post<Draft>`; that's a compile error, not a runtime check. `PhantomData<State>`
carries the type parameter without storing any actual data of that type
(the struct's runtime layout is identical regardless of `State`). This is
the pattern for state machines where an invalid transition should be
impossible to write, not just impossible to reach at runtime.

## Rust-specific traps

**Builder methods that take `&mut self` instead of `self`.** Returning
`&mut Self` chains fine but forces the builder to live in a `let mut`
variable and forces `build()` to take `&self`, which then can't move fields
out without cloning. Taking `self` by value (as above) is more idiomatic —
each step consumes the builder and hands back a new one.

**`dyn Trait` needs `Sized`-free methods.** A trait with a method returning
`Self` or taking `Self` by value can't be made into a trait object at all —
the compiler rejects `Box<dyn Discount>` with "the trait cannot be made into
an object" if `Discount` ever grows a method like `fn combine(self, other:
Self) -> Self`. This is discovered late, often after a trait has many
implementors, forcing an API redesign.

**Typestate transitions that borrow instead of consume.** `fn publish(&self)
-> Post<Published>` (borrowing instead of taking `self`) lets the original
`Draft` value keep existing and being published again — the type-level
guarantee ("a post is published exactly once, from a draft") silently
evaporates. `fn publish(self)` is required for the trick to actually enforce
anything; the whole pattern rests on ownership being consumed.

**Newtype boilerplate for arithmetic.** `Cents` above only implements
`Display`; the moment you want `Cents + Cents` or `Cents * u32`, you're
manually implementing `std::ops::Add`/`Mul` per operation — there's no
`#[derive]` for arithmetic traits, unlike `Debug` or `Clone`.

## Cheat sheet

| Pattern | Rust mechanism |
|---|---|
| Builder | Chained `self -> Self` methods, final `build(self) -> T` |
| Strategy | `&dyn Trait` or generic `impl Trait` parameter |
| Newtype | Tuple struct wrapping one field, own trait impls |
| Typestate | Generic struct parameterized by marker types + `PhantomData` |
| Singleton (rare in Rust) | `once_cell`/`std::sync::OnceLock`, not a class-level static |
| Observer | Channel (`mpsc`) or callback `Vec<Box<dyn Fn(...)>>` |

## Exercise

Extend the typestate example with a third state, `Archived`, reachable only
from `Post<Published>` via an `archive(self) -> Post<Archived>` method.
Give `Post<Archived>` its own method, `restore(self) -> Post<Draft>`, and
verify in `main` that a `Post<Draft>` cannot call `archive()` directly —
paste the exact compiler error you get when you try.

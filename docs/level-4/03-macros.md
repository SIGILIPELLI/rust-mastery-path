# 03 · Macros

Rust has two macro systems that solve different problems. `macro_rules!`
(declarative macros) pattern-match on token trees and substitute — good for
"I keep writing this same shape of code with different values inside it."
Proc macros run arbitrary Rust code at compile time against a real syntax
tree — what `#[derive(Debug)]`, `#[derive(Serialize)]`, and every other
derive you've used are built from. This module builds one of each.

## Declarative macros: pattern + repetition

```rust
macro_rules! hashmap {
    () => { std::collections::HashMap::new() };
    ( $( $key:expr => $val:expr ),* $(,)? ) => {{
        let mut map = std::collections::HashMap::new();
        $( map.insert($key, $val); )*
        map
    }};
}

fn main() {
    let scores = hashmap! {
        "alice" => 90,
        "bob" => 85,
    };
    let mut keys: Vec<_> = scores.keys().collect();
    keys.sort();
    for k in keys {
        println!("{k}: {}", scores[k]);
    }
}
```

```text
alice: 90
bob: 85
```

`$( $key:expr => $val:expr ),* $(,)?` reads as: zero or more
`expr => expr` pairs, comma-separated, with an optional trailing comma. Each
`$key:expr` is a *fragment specifier* — `expr` here means "match anything
that parses as a Rust expression," which is why `hashmap!{ 1+1 => f() }`
works without special-casing arithmetic or function calls. The macro has
two arms: the empty case, and the repeating case — `macro_rules!` dispatches
on which pattern the input token stream matches, same idea as `match`.

## Hygiene: a macro's internal names don't leak

```rust
macro_rules! swap_via_tmp {
    ($a:expr, $b:expr) => {{
        let tmp = $a;
        $a = $b;
        $b = tmp;
    }};
}

fn main() {
    let tmp = 999; // caller's own `tmp` -- must not collide with the macro's
    let mut a = 1;
    let mut b = 2;
    swap_via_tmp!(a, b);
    println!("a={a} b={b} tmp={tmp}");
}
```

```text
a=2 b=1 tmp=999
```

The caller's `tmp` survives untouched even though the macro also declares a
variable named `tmp`. This is *macro hygiene*: identifiers introduced inside
a `macro_rules!` expansion live in their own syntactic context and don't
collide with identically-named identifiers at the call site, even though
both literally say `tmp`. This is exactly the trap C's text-substitution
macros fall into — Rust's macro system was designed specifically to not have
that class of bug.

## Repetition with a running computation

```rust
macro_rules! max_of {
    ($first:expr $(, $rest:expr)*) => {{
        let mut m = $first;
        $( if $rest > m { m = $rest; } )*
        m
    }};
}

fn main() {
    println!("max: {}", max_of!(3, 7, 2, 9, 4));
}
```

```text
max: 9
```

`max_of!` takes a variable number of arguments and expands into ordinary
sequential comparisons — no runtime loop over a collection, just repeated
`if` statements generated at compile time, one per argument. This is a
pattern `macro_rules!` can do that a normal generic function can't: the
number of arguments is baked into the expansion itself.

## Proc macros: a real `#[derive(Describe)]`

Declarative macros work on token patterns; a derive proc macro gets a full
parsed AST (via the `syn` crate) and builds new code with `quote!`. This
needs its own crate with `proc-macro = true`:

```toml
# describe_derive/Cargo.toml
[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    let name_str = name.to_string();

    let field_count = match input.data {
        Data::Struct(ref data) => match &data.fields {
            Fields::Named(f) => f.named.len(),
            Fields::Unnamed(f) => f.unnamed.len(),
            Fields::Unit => 0,
        },
        _ => 0,
    };

    let expanded = quote! {
        impl #name {
            pub fn describe() -> String {
                format!("{} has {} field(s)", #name_str, #field_count)
            }
        }
    };

    expanded.into()
}
```

Used from a separate crate:

```rust
use describe_derive::Describe;

#[derive(Describe)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    println!("{}", Point::describe());
}
```

```text
Point has 2 field(s)
```

`parse_macro_input!` turns the raw `TokenStream` the compiler hands the
macro into a `syn::DeriveInput` — a real, typed representation of "here's a
struct/enum named X with these fields." `quote!` does the reverse:
Rust-code-as-text templating where `#name` and `#field_count` splice values
from the surrounding Rust scope into the generated tokens, which then get
handed back to the compiler as if you'd typed them yourself.

## Rust-specific traps

**`macro_rules!` must be defined before use in the same file (or imported).**
Unlike functions, which can be called before their definition appears later
in the file, a `macro_rules!` macro is only visible textually after its
`macro_rules!` block, top to bottom, unless `#[macro_export]` and an
explicit `use` handle it across modules.

**Proc macro crates cannot contain anything except proc macros.** A crate
with `proc-macro = true` can only export `#[proc_macro]`,
`#[proc_macro_derive]`, and `#[proc_macro_attribute]` functions — no regular
`pub fn`, no shared types re-exported for consumers to use directly. Shared
logic between the macro and its users typically lives in a third, plain
crate that both depend on.

**Fragment specifiers restrict what comes after them.** `$e:expr` in
`macro_rules!` can only be followed by a small fixed set of tokens (`=>`,
`,`, `;`, and a few others) — you cannot write
`($e:expr foo)` because the macro system needs to know unambiguously
where the expression match ends. This produces a "local ambiguity" or
"no rules expected this token" compile error that has nothing to do with
your actual logic.

**Debugging expansion needs `cargo expand`.** Neither declarative nor proc
macro errors show you the *expanded* code by default — a type error inside
generated code reports the macro invocation's span, not the generated
line. `cargo install cargo-expand` and `cargo expand` print what the macro
actually produced, which is usually the fastest way to find the real bug.

## Cheat sheet

| | `macro_rules!` | proc macro |
|---|---|---|
| Input | Token patterns you define | Full parsed AST (`syn::DeriveInput` etc.) |
| Crate requirement | None — lives in any crate | Separate crate with `proc-macro = true` |
| Typical use | Repetitive syntax, small DSLs | `#[derive(...)]`, custom attributes |
| Hygiene | Automatic (caller/macro names don't collide) | Manual — you construct identifiers yourself |
| Debugging | `cargo expand`, careful pattern reading | `cargo expand`, `eprintln!` inside the macro fn |

## Exercise

Extend `describe_derive` to also count fields for enums (currently
`field_count` is `0` for anything that isn't `Data::Struct`) — for an enum,
report the number of *variants* instead of fields, with a different message
format (`"{} has {} variant(s)"`). Derive `Describe` on an enum with three
variants and confirm the printed message is correct.

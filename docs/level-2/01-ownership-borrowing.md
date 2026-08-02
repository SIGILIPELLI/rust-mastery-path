# 01 · Ownership & Borrowing Deep Dive

Level 1 introduced ownership as "every value has one owner, and assignment
moves non-`Copy` values." That's true, but it only gets you so far — the
moment you want two parts of a program to look at the same data without
constantly moving or cloning it, you need **borrowing**: references that let
code read (or even modify) a value without taking ownership of it. This
module covers the actual rules the borrow checker enforces, why they exist,
and the specific ways beginners get stuck fighting the compiler.

## Why borrowing exists

Without it, sharing data would force a choice between two bad options: move
ownership everywhere (so only one function can ever touch a value again) or
clone everywhere (paying a runtime cost just to read something). References
give you a third option — temporary, checked access — with zero runtime
overhead, because all the checking happens at compile time.

```rust
fn print_length(s: &String) {
    // `s` is a reference -- this function borrows, it doesn't own
    println!("length: {}", s.len());
}

fn main() {
    let name = String::from("Ferris");
    print_length(&name);   // pass a reference with `&`
    print_length(&name);   // fine -- name is still owned by main, can be borrowed again

    println!("{name}");    // still valid -- ownership never left main
}
```

```text
length: 6
length: 6
Ferris
```

Compare this to passing `name` by value: the function would take ownership,
and the second call plus the final `println!` would both fail to compile.

## The rule: many readers, or one writer, never both

This is the single rule that generates most "fighting the borrow checker"
moments, so it's worth stating precisely: at any point in the code, for a
given value, you may have **either** any number of immutable references
(`&T`) **or exactly one** mutable reference (`&mut T`) — never both kinds at
the same time.

```rust
fn main() {
    let mut score = 10;

    let r1 = &score;   // immutable borrow
    let r2 = &score;   // another immutable borrow -- fine, reading is safe to share
    println!("{r1} {r2}");   // 10 10

    let r3 = &mut score;   // mutable borrow -- only allowed once r1/r2 are done being used
    *r3 += 5;
    println!("{r3}");   // 15
}
```

The compiler tracks a reference's **last use**, not its scope — `r1` and
`r2`'s last use is the `println!` above, so `r3` is allowed afterward. This
is called *non-lexical lifetimes*, and it's why the rule feels less rigid
than "no mixing borrows in the same block."

```rust
fn main() {
    let mut score = 10;

    let r1 = &score;
    let r3 = &mut score;   // ERROR: cannot borrow `score` as mutable
                            // because it is also borrowed as immutable
    println!("{r1} {r3}");
}
```

```text
error[E0502]: cannot borrow `score` as mutable because it is also
borrowed as immutable
```

Here `r1` is still alive (it gets used in the final `println!`), so the
mutable borrow on the next line overlaps with it — that's the conflict the
compiler rejects.

## Why the rule exists: preventing data races at compile time

The "one writer XOR many readers" rule is exactly the condition that
prevents data races, generalized to single-threaded code too: if two parts
of the code can both see a value and one of them can change it out from
under the other, you get bugs that are notoriously hard to reproduce
(iterator invalidation, stale reads, use-after-free in other languages).
Rust makes this a compile error instead of a runtime surprise.

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    for item in &v {
        // v.push(4);
        // ERROR: cannot borrow `v` as mutable because it is also
        // borrowed as immutable (by the `for` loop's iterator)
        println!("{item}");
    }

    v.push(4);   // fine -- the immutable borrow from the loop has ended
    println!("{:?}", v);   // [1, 2, 3, 4]
}
```

The commented-out `v.push(4)` would resize the vector while iterating over
it — in a language without this check, that can invalidate the iterator and
either panic or read garbage. Rust catches it before the program ever runs.

## Mutable references must be exclusive

```rust
fn add_one(n: &mut i32) {
    *n += 1;   // `*` dereferences to get at the value behind the reference
}

fn main() {
    let mut x = 5;
    add_one(&mut x);
    add_one(&mut x);
    println!("{x}");   // 7
}
```

```text
7
```

Only one `&mut` to a value can exist at a time, and it can't coexist with
any `&`. This isn't a restriction to work around — it's what lets the
compiler (and you, reading the code) reason locally: if you hold `&mut x`,
you know with certainty nothing else can read or write `x` while you have
it.

## Dangling references are impossible

A classic C/C++ bug is returning a pointer to a local variable that gets
freed when the function returns. Rust's borrow checker rejects this at
compile time via **lifetimes** (the full topic of the next module) — here's
the shape of the error you'd see:

```rust
fn dangle() -> &String {          // ERROR: missing lifetime specifier
    let s = String::from("hi");
    &s
    // `s` is dropped here -- the reference would point at freed memory
}
```

```text
error[E0106]: missing lifetime specifier
help: this function's return type contains a borrowed value, but
there is no value for it to be borrowed from
```

The fix isn't a lifetime annotation — it's to return the owned value
instead, letting ownership move out of the function:

```rust
fn no_dangle() -> String {
    let s = String::from("hi");
    s   // ownership moves out -- no reference, nothing left behind to dangle
}

fn main() {
    let s = no_dangle();
    println!("{s}");   // hi
}
```

## Borrowing struct fields independently

The borrow checker (since the 2021 edition's "disjoint closure captures" and
general field-level borrow tracking) understands that borrowing two
*different* fields of the same struct at once is fine, even though it looks
like "two mutable borrows":

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let mut p = Point { x: 1, y: 2 };

    let x_ref = &mut p.x;
    let y_ref = &mut p.y;   // fine -- disjoint fields, not the same data

    *x_ref += 10;
    *y_ref += 20;
    println!("{}, {}", p.x, p.y);   // 11, 22
}
```

This only works for direct field access, not through a method call — a
method taking `&mut self` borrows the *whole* struct, because the compiler
can't see inside the method to know it only touches one field.

## Slices: borrowing part of a collection

A slice (`&[T]` or `&str`) is a reference to a contiguous run of elements
without owning them — it's how you borrow "part of" a `Vec` or `String`.

```rust
fn main() {
    let numbers = vec![10, 20, 30, 40, 50];
    let middle: &[i32] = &numbers[1..4];   // borrow indices 1, 2, 3
    println!("{:?}", middle);   // [20, 30, 40]

    let text = String::from("hello world");
    let first_word: &str = &text[0..5];   // borrow just "hello"
    println!("{first_word}");   // hello
}
```

```text
[20, 30, 40]
hello
```

Slices are why `fn print_length(s: &str)` is generally preferred over
`fn print_length(s: &String)` for function parameters: a `&str` accepts a
borrow of a `String`, a string literal, or a slice of either, while `&String`
only accepts the first.

## Cheat sheet

| Situation | Allowed? |
|-----------|----------|
| Multiple `&T` to the same value | Yes |
| One `&mut T` to the same value | Yes |
| `&T` and `&mut T` to the same value at once | No |
| Two `&mut T` to the same value at once | No |
| `&mut` to two different fields of the same struct | Yes |
| Returning a reference to a local variable | No (dangling) — return owned data instead |
| Reference outliving the data it points to | No — caught at compile time |

## Exercise

Write a function `fn longest_word(text: &str) -> &str` that returns the
longest whitespace-separated word in `text` as a slice (no cloning). Then
write a function `fn shout(word: &mut String)` that appends `"!"` to a
mutable `String` in place. In `main`, call `longest_word` on a sentence and
print the result, then build a `String`, pass it to `shout` with `&mut`, and
print it afterward to confirm the mutation stuck. Finally, try (in a
comment) writing code that holds an immutable reference from `longest_word`
across a call to `shout` on the same string, and explain in a comment why
the compiler would reject it.

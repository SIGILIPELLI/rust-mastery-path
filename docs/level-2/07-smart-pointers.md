# 07 · Smart Pointers (Box, Rc, RefCell)

A smart pointer is a struct that behaves like a reference but adds extra
capabilities — heap allocation, shared ownership, or runtime-checked
mutability. `String` and `Vec<T>` are smart pointers you've already used
without the label. This module covers the three you'll reach for
deliberately: `Box<T>` for heap allocation, `Rc<T>` for sharing ownership
between multiple parts of a program, and `RefCell<T>` for mutating data
behind a reference that's normally immutable — plus how `Rc` and `RefCell`
combine, since that combination shows up constantly in real Rust code.

## `Box<T>` — putting a value on the heap

```rust
fn main() {
    let x = Box::new(5);   // 5 is heap-allocated, x holds a pointer to it
    println!("{x}");       // 5 -- Box derefs automatically, no `*` needed here

    // Box lets you use a value where "the exact size must be known at
    // compile time" would otherwise fail -- a Box is always pointer-sized
    // regardless of what it points to.
}
```

```text
5
```

## Why `Box` is required for recursive types

A `struct` or `enum` that contains itself directly has no fixed size — the
compiler would need to know the size of `List`, which needs the size of
`List`, forever. `Box<T>` breaks the cycle because a `Box` is always one
pointer wide, no matter what it points to:

```rust
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),   // Box breaks the infinite-size recursion
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    println!("{:?}", list);
}
```

```text
Cons(1, Cons(2, Cons(3, Nil)))
```

Without `Box`, `enum List { Cons(i32, List), Nil }` fails to compile with
"recursive type has infinite size" — `Box<List>` is the standard fix any
time you build a linked list, tree, or other self-referential structure.

## `Rc<T>` — shared ownership via reference counting

Ownership rules say one value has exactly one owner — but sometimes several
parts of a program genuinely need to share the same data (a tree node with
multiple parents, a graph, a cache referenced from several places). `Rc<T>`
("reference counted") allows multiple owners by tracking how many `Rc`
handles point at the same heap value, freeing it only when the count hits
zero.

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("shared data"));
    println!("count after creating a: {}", Rc::strong_count(&a));   // 1

    let b = Rc::clone(&a);   // NOT a deep copy -- just increments the count
    println!("count after cloning into b: {}", Rc::strong_count(&a));   // 2

    {
        let c = Rc::clone(&a);
        println!("count after cloning into c: {}", Rc::strong_count(&a));   // 3
    }   // c drops here, count decreases

    println!("count after c goes out of scope: {}", Rc::strong_count(&a));   // 2
    println!("{a} / {b}");   // both still point at the same data
}
```

```text
count after creating a: 1
count after cloning into b: 2
count after cloning into c: 3
count after c goes out of scope: 2
shared data / shared data
```

`Rc::clone(&a)` is cheap — it's a pointer copy plus an increment, not a copy
of the underlying `String`. This is why `Rc::clone` is idiomatic over
`a.clone()` even though both work: writing `Rc::clone` signals to a reader
"this is just a reference-count bump," not a potentially expensive deep
copy.

`Rc<T>` only works single-threaded — sharing across threads needs `Arc<T>`
("atomic Rc"), which is otherwise identical but uses thread-safe atomic
operations for the count.

## `Rc<T>` only allows shared *immutable* access

```rust
use std::rc::Rc;

fn main() {
    let shared = Rc::new(5);
    let a = Rc::clone(&shared);

    // *a += 1;
    // ERROR: cannot assign to data in an `Rc` -- Rc only gives you `&T`,
    // never `&mut T`, because it can't guarantee you're the only owner.

    println!("{a}");
}
```

This is the gap `RefCell<T>` fills.

## `RefCell<T>` — mutability checked at runtime instead of compile time

The borrow checker's rules (Module 01) are enforced at compile time, which
is normally exactly what you want. But sometimes the *shape* of a valid
program doesn't fit the compiler's static analysis — most commonly, when
you have an `Rc<T>` (shared, immutable access) but still need to mutate the
shared data. `RefCell<T>` moves the borrow-checking rule from compile time
to runtime: it still enforces "one mutable borrow XOR many immutable
borrows," but by panicking at runtime instead of refusing to compile.

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    *cell.borrow_mut() += 10;   // mutable borrow, checked at runtime
    println!("{}", cell.borrow());   // 15

    let r1 = cell.borrow();
    let r2 = cell.borrow();   // multiple immutable borrows -- fine
    println!("{} {}", r1, r2);
}
```

```text
15
15 15
```

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    let _r1 = cell.borrow();
    let _r2 = cell.borrow_mut();   // PANICS at runtime, not a compile error
    // thread 'main' panicked at ...: RefCell already borrowed
}
```

This trade-off is deliberate: `RefCell` moves a check the compiler *could*
have rejected safely, but couldn't prove safe with its static rules, to
runtime — you gain flexibility, but a bug that would have been a compile
error with a plain reference now becomes a panic you only see when that
code path actually runs.

## `Rc<RefCell<T>>` — the combination you'll actually use

Shared ownership (`Rc`) plus interior mutability (`RefCell`) together give
you what plain references can't: multiple owners that can each mutate the
shared data.

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Counter {
    count: i32,
}

fn main() {
    let shared_counter = Rc::new(RefCell::new(Counter { count: 0 }));

    let handle_a = Rc::clone(&shared_counter);
    let handle_b = Rc::clone(&shared_counter);

    handle_a.borrow_mut().count += 1;
    handle_b.borrow_mut().count += 1;
    handle_a.borrow_mut().count += 1;

    println!("{:?}", shared_counter.borrow());
    // Counter { count: 3 } -- all three handles mutated the same data
}
```

```text
Counter { count: 3 }
```

This pattern (multiple `Rc<RefCell<T>>` handles pointing at the same node)
is exactly how tree and graph structures with shared, mutable nodes are
built in Rust, since a plain `&mut` reference to a shared node isn't
possible under the compile-time borrow rules.

## Choosing between them

| Type | Ownership | Mutability | Cost | Use when |
|------|-----------|------------|------|----------|
| `Box<T>` | Single owner | Same as the inner value | One heap allocation | Recursive types, or moving a large value to the heap |
| `Rc<T>` | Multiple owners | Immutable only | Reference-count increment/decrement | Multiple parts of the program need to share read-only data |
| `RefCell<T>` | Single owner | Interior mutability, runtime-checked | Runtime borrow-flag check | You need to mutate through what looks like an immutable reference |
| `Rc<RefCell<T>>` | Multiple owners | Interior mutability, runtime-checked | Both of the above | Shared data that multiple owners all need to mutate |

## Exercise

Model a simple shared shopping cart: define
`struct Cart { items: Vec<String>, total: f64 }`. Wrap it in
`Rc<RefCell<Cart>>` and create two handles (`Rc::clone`) representing two
parts of a program that both add items. From each handle, call
`.borrow_mut()` to push an item name and add its price to `total`. After
both have added an item, print the final cart through a third `.borrow()`
call, and print `Rc::strong_count` at each step to confirm it goes 1 -> 2 ->
3 as you create handles.

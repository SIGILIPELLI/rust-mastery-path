# 04 · Closures & Iterators

Closures are anonymous functions that can capture variables from the scope
they're defined in; iterators are Rust's abstraction for "a sequence of
values, produced one at a time." The two are deeply linked — almost every
useful iterator method takes a closure — and together they let you write
data-processing pipelines that read like a description of *what* you want,
while compiling down to code as fast as a hand-written loop.

## Basic closure syntax

```rust
fn main() {
    let add_one = |x: i32| x + 1;         // full syntax
    let add_two = |x| x + 2;              // types inferred from usage

    println!("{}", add_one(5));   // 6
    println!("{}", add_two(5));   // 7

    // Multi-statement closures use a block body
    let describe = |n: i32| {
        let parity = if n % 2 == 0 { "even" } else { "odd" };
        format!("{n} is {parity}")
    };
    println!("{}", describe(7));   // 7 is odd
}
```

```text
6
7
7 is odd
```

Unlike `fn`, closures can omit parameter and return types — the compiler
infers them from how the closure is used, the first time it's called. This
also means a single closure can't be called with two different argument
types later, unlike a generic function.

## Capturing the environment

The feature that makes closures more than "functions without type
annotations" is that they can capture variables from their surrounding
scope:

```rust
fn main() {
    let factor = 3;
    let multiply = |x: i32| x * factor;   // captures `factor` by reference

    println!("{}", multiply(4));   // 12
    println!("{}", multiply(5));   // 15

    // A closure that mutates a captured variable must be `mut` and `FnMut`
    let mut total = 0;
    let mut accumulate = |x: i32| {
        total += x;   // mutably borrows `total`
        println!("running total: {total}");
    };
    accumulate(10);
    accumulate(5);
}
```

```text
12
15
running total: 10
running total: 15
```

## `Fn`, `FnMut`, `FnOnce` — the three closure traits

Every closure implements one or more of these traits, based on *how* it uses
its captured variables. You rarely name these traits yourself for simple
code, but they're exactly what a function signature like
`fn apply<F: Fn(i32) -> i32>(f: F)` is constraining, and the compiler's error
messages reference them directly, so recognizing them matters:

| Trait | Can be called | Captures by |
|-------|----------------|-------------|
| `Fn` | Any number of times | Reference (`&T`) — doesn't consume or mutate what it captures |
| `FnMut` | Any number of times | Mutable reference (`&mut T`) — can mutate captured state |
| `FnOnce` | Exactly once | By value — consumes (moves) what it captures |

```rust
fn call_with_one<F: Fn(i32) -> i32>(f: F) -> i32 {
    f(1)
}

fn call_and_mutate<F: FnMut()>(mut f: F) {
    f();
    f();
}

fn call_once<F: FnOnce() -> String>(f: F) -> String {
    f()   // can only be called once -- fine, we only call it once here
}

fn main() {
    let double = |x: i32| x * 2;
    println!("{}", call_with_one(double));   // 2

    let mut count = 0;
    call_and_mutate(|| {
        count += 1;
        println!("count = {count}");
    });

    let name = String::from("Ferris");
    let consume = move || format!("Hello, {name}!");   // `move` forces capture by value
    println!("{}", call_once(consume));
}
```

```text
2
count = 1
count = 2
Hello, Ferris!
```

`move` forces the closure to take ownership of everything it captures,
instead of borrowing — essential when the closure needs to outlive the
scope it was created in (like when it's sent to another thread), and the
reason `consume` above is only callable once: it owns `name`, and calling it
moves that ownership out.

## The `Iterator` trait and laziness

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let iter = numbers.iter().map(|x| x * 2);
    // Nothing has been computed yet -- `map` is lazy, it just wraps the
    // iterator with a description of the transformation.

    println!("about to consume");
    let doubled: Vec<i32> = iter.collect();   // NOW the map closure actually runs
    println!("{:?}", doubled);
}
```

```text
about to consume
[2, 4, 6, 8, 10]
```

This laziness is a common trap for people coming from languages where `map`
runs immediately: an iterator adapter chain does *nothing* until you call a
consuming method (`.collect()`, `.sum()`, `.for_each()`, a `for` loop, etc.).
If you build a chain and never consume it, the compiler will warn that it's
unused — the computation genuinely never happened.

## Iterator adapters: map, filter, fold

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // map -- transform each element
    let squared: Vec<i32> = numbers.iter().map(|n| n * n).collect();
    println!("{:?}", squared);
    // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

    // filter -- keep elements matching a predicate
    let evens: Vec<&i32> = numbers.iter().filter(|n| **n % 2 == 0).collect();
    println!("{:?}", evens);
    // [2, 4, 6, 8, 10]

    // fold -- reduce to a single value, given a starting accumulator
    let sum = numbers.iter().fold(0, |acc, n| acc + n);
    println!("{sum}");   // 55

    // Chaining -- filter, then map, then collect -- reads like the intent
    let sum_of_even_squares: i32 = numbers
        .iter()
        .filter(|n| **n % 2 == 0)
        .map(|n| n * n)
        .sum();
    println!("{sum_of_even_squares}");   // 220
}
```

```text
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
[2, 4, 6, 8, 10]
55
220
```

Note the `**n` in `filter` — `.iter()` yields `&i32`, and `.filter`'s
closure receives a reference *to* that item (`&&i32`), so one `*` gets back
to `&i32` and the second gets to `i32`. This double-reference is one of the
most common "why won't this compile" moments with iterators; the fix is
almost always an extra `*`, or switching to `.copied()` right after `.iter()`
to work with plain values instead of references.

## More adapters worth knowing

```rust
fn main() {
    let words = vec!["rust", "is", "fun"];

    // enumerate -- pair each item with its index
    for (i, word) in words.iter().enumerate() {
        println!("{i}: {word}");
    }

    // zip -- pair up items from two iterators
    let lengths: Vec<usize> = words.iter().map(|w| w.len()).collect();
    for (word, len) in words.iter().zip(lengths.iter()) {
        println!("{word} has {len} letters");
    }

    // take / skip -- limit or skip a number of items
    let first_two: Vec<&&str> = words.iter().take(2).collect();
    println!("{:?}", first_two);   // ["rust", "is"]

    // any / all -- short-circuiting boolean checks
    println!("{}", words.iter().any(|w| w.len() > 3));   // true ("rust")
    println!("{}", words.iter().all(|w| w.len() > 1));   // true
}
```

```text
0: rust
1: is
2: fun
rust has 4 letters
is has 2 letters
fun has 3 letters
["rust", "is"]
true
true
```

## Cheat sheet

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

## Exercise

Given `let words = vec!["apple", "kiwi", "banana", "fig", "cherry"];`, write
an iterator chain that: filters to words longer than 3 letters, maps each to
its uppercase form (`.to_uppercase()`), and collects the result into a
`Vec<String>`. Print it. Then write a function
`fn make_multiplier(factor: i32) -> impl Fn(i32) -> i32` that returns a
closure capturing `factor`, use it to create a `times_three` closure, and
apply it to every element of a `Vec<i32>` using `.map()`.

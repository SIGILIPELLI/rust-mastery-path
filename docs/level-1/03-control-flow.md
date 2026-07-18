# 03 · Control Flow

Rust's control flow will feel familiar if you've used any C-family language,
with one big difference: `if` and `loop` are **expressions** — they can
produce a value you bind directly to a variable, not just branch execution.

## `if` / `else if` / `else`

```rust
fn main() {
    let temperature = 15;

    if temperature > 30 {
        println!("It's hot");
    } else if temperature > 10 {
        println!("It's mild");
    } else {
        println!("It's cold");
    }
    // It's mild
}
```

Note there are no parentheses required around the condition (unlike C/Java),
but the braces are mandatory even for a single statement.

## `if` as an expression

Because `if`/`else` evaluates to a value, you can use it directly in a `let`
binding — as long as every branch produces the same type:

```rust
fn main() {
    let temperature = 15;

    let description = if temperature > 30 {
        "hot"
    } else if temperature > 10 {
        "mild"
    } else {
        "cold"
    };

    println!("It's {description}");   // It's mild
}
```

Each branch is an expression with no trailing semicolon — that's what makes
its value "returned" from the block into `description`.

## `loop` — an infinite loop that can return a value

`loop` repeats forever until you `break` out of it. `break` can carry a value,
which becomes the result of the whole `loop` expression.

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;   // stops the loop and yields this value
        }
    };

    println!("result = {result}");   // result = 20
}
```

This pattern is handy for retry logic or computations that need a few
iterations before a value is ready.

## `while`

`while` repeats as long as a condition holds — no value production, just
repetition.

```rust
fn main() {
    let mut countdown = 3;

    while countdown > 0 {
        println!("{countdown}...");
        countdown -= 1;
    }
    println!("Liftoff!");
    // 3...
    // 2...
    // 1...
    // Liftoff!
}
```

## `for` with ranges

`for` is the idiomatic way to iterate a known number of times or over a
collection. `0..5` is a **half-open range** (excludes 5); `0..=5` is
**inclusive** (includes 5).

```rust
fn main() {
    for i in 0..5 {
        print!("{i} ");
    }
    println!();
    // 0 1 2 3 4

    for i in 0..=5 {
        print!("{i} ");
    }
    println!();
    // 0 1 2 3 4 5
}
```

## `for` over a collection

Iterating a `Vec` or array with `.iter()` gives you references to each
element without taking ownership of the collection.

```rust
fn main() {
    let scores = vec![90, 85, 77, 100];

    for score in scores.iter() {
        println!("Score: {score}");
    }
    // Score: 90
    // Score: 85
    // Score: 77
    // Score: 100

    // `for x in &collection` is equivalent shorthand for `.iter()`
    let names = ["Alice", "Bob", "Cara"];
    for name in &names {
        println!("Hi, {name}");
    }
    // Hi, Alice
    // Hi, Bob
    // Hi, Cara
}
```

`Vec<T>` and other collection basics get their own module — see
[Module 6](06-collections.md) — but this much is enough to loop over one.

## `continue`

`continue` skips the rest of the current iteration and moves to the next one.

```rust
fn main() {
    for i in 1..=10 {
        if i % 2 != 0 {
            continue;   // skip odd numbers
        }
        print!("{i} ");
    }
    println!();
    // 2 4 6 8 10
}
```

## Cheat sheet

| Construct | Produces a value? | Typical use |
|-----------|--------------------|-------------|
| `if`/`else if`/`else` | Yes, can be used in `let` | Branching, conditional value |
| `loop` | Yes, via `break value` | Repeat until a condition inside is met |
| `while` | No | Repeat while a condition holds |
| `for x in 0..n` | No | Repeat a known number of times |
| `for x in collection.iter()` | No | Visit every element of a collection |
| `continue` | — | Skip to the next loop iteration |
| `break` | Optional value (in `loop`) | Exit a loop early |

## Exercise

Write a program that uses a `for` loop over `1..=20` to print `"Fizz"` for
multiples of 3, `"Buzz"` for multiples of 5, `"FizzBuzz"` for multiples of
both, and the number itself otherwise (skip printing anything extra using
`continue` where it helps you keep the logic clean). Then write a `loop` that
starts a counter at 1, doubles it each iteration, and `break`s with the first
value greater than 100, printing that final value.

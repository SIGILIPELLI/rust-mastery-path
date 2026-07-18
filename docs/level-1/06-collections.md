# 06 · Collections (Vec, String, HashMap)

So far you've worked with single values and fixed-size arrays. Real programs
need collections that can grow: an ordered, resizable list (`Vec<T>`), a
growable piece of text (`String`), and a lookup table of key-value pairs
(`HashMap<K, V>`). All three live on the heap, which is exactly why they can
grow — the compiler doesn't need to know their size ahead of time.

## `Vec<T>` — a growable list

```rust
fn main() {
    // Two ways to create a Vec:
    let mut numbers: Vec<i32> = Vec::new();   // empty, type annotated
    numbers.push(10);
    numbers.push(20);
    numbers.push(30);

    let mut fruits = vec!["apple", "banana", "cherry"]; // vec! macro, type inferred

    println!("numbers = {:?}", numbers);
    println!("fruits = {:?}", fruits);

    // Indexing (panics if out of bounds)
    println!("first fruit: {}", fruits[0]);

    // .get() returns Option<&T> instead of panicking
    match fruits.get(10) {
        Some(f) => println!("found: {f}"),
        None => println!("no fruit at index 10"),
    }

    // pop() removes and returns the last element, wrapped in Option
    let last = numbers.pop();
    println!("popped: {:?}", last);   // Some(30)
    println!("numbers now = {:?}", numbers);

    // len() and iterating
    println!("fruits has {} items", fruits.len());
    for fruit in fruits.iter() {
        println!("- {fruit}");
    }

    // Mutating while iterating with iter_mut()
    for fruit in fruits.iter_mut() {
        *fruit = if *fruit == "banana" { "BANANA" } else { fruit };
    }
    println!("fruits = {:?}", fruits);
}
```

```text
numbers = [10, 20, 30]
fruits = ["apple", "banana", "cherry"]
first fruit: apple
no fruit at index 10
popped: Some(30)
numbers now = [10, 20]
fruits has 3 items
- apple
- banana
- cherry
fruits = ["apple", "BANANA", "cherry"]
```

`fruits.iter()` borrows each element as `&T` so the `Vec` is still usable
afterward; a plain `for fruit in fruits` (no `.iter()`) would *move* the `Vec`
and consume it, which only matters once the elements aren't a `Copy` type
like `&str`.

## `String` vs `&str`

Rust has two string types, and the difference matters constantly:

- **`&str`** — a *borrowed* view into string data (a "string slice"). String
  literals like `"hello"` are `&str`.
- **`String`** — an *owned*, growable, heap-allocated string.

```rust
fn main() {
    let borrowed: &str = "hello";          // borrowed, fixed
    let owned: String = String::from("hello"); // owned, growable
    let also_owned: String = "hello".to_string(); // another way to own it

    // Concatenation with +  (consumes the left-hand String!)
    let greeting = String::from("Hello, ") + "world" + "!";
    println!("{greeting}");

    // format! is usually nicer -- doesn't consume anything, works like println!
    let name = "Ferris";
    let message = format!("Hello, {name}! You are {} years old.", 10);
    println!("{message}");

    // Growing a String in place
    let mut sentence = String::from("Rust");
    sentence.push_str(" is fun");
    sentence.push('!');
    println!("{sentence}");

    // Iterating over characters
    for c in "Ferris".chars() {
        print!("{c}-");
    }
    println!();

    // A &str borrowed from a String (slicing)
    let slice: &str = &sentence[0..4];
    println!("first 4 bytes: {slice}");

    let _ = borrowed;
    let _ = owned;
    let _ = also_owned;
}
```

```text
Hello, world!
Hello, Ferris! You are 10 years old.
Rust is fun!
F-e-r-r-i-s-
first 4 bytes: Rust
```

A good rule of thumb while learning: use `&str` for function parameters when
you only need to *read* text, and `String` when you need to *own or build*
text. Slicing a `String` (`&sentence[0..4]`) works on byte indices, so be
careful with non-ASCII text — that's a subtlety you'll revisit later, not
something to worry about yet.

## `HashMap<K, V>` — key-value lookup

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();

    scores.insert(String::from("Alice"), 90);
    scores.insert(String::from("Bob"), 85);
    scores.insert(String::from("Alice"), 95); // overwrites the previous value

    // .get() returns Option<&V> -- there's no guarantee the key exists
    match scores.get("Alice") {
        Some(score) => println!("Alice scored {score}"),
        None => println!("Alice has no score"),
    }

    match scores.get("Charlie") {
        Some(score) => println!("Charlie scored {score}"),
        None => println!("Charlie has no score yet"),
    }

    // entry().or_insert() -- insert a default only if the key is missing
    scores.entry(String::from("Charlie")).or_insert(0);
    *scores.entry(String::from("Bob")).or_insert(0) += 10; // bump Bob's score

    // Iterating (order is NOT guaranteed with HashMap)
    let mut pairs: Vec<(&String, &i32)> = scores.iter().collect();
    pairs.sort(); // sort so the output below is deterministic
    for (name, score) in pairs {
        println!("{name}: {score}");
    }

    println!("total entries: {}", scores.len());
}
```

```text
Alice scored 95
Charlie has no score yet
Alice: 95
Bob: 95
Charlie: 0
total entries: 3
```

The `entry().or_insert()` pattern is the idiomatic way to say "give me a
mutable handle to this key's value, inserting a default first if it isn't
there yet" — it's the classic tool for counting things, like tallying how
many times each word appears in a block of text.

## Cheat sheet

| Type | Owns its data? | Grows? | Common methods |
|------|-----------------|--------|-----------------|
| `Vec<T>` | Yes | Yes | `.push()`, `.pop()`, `.get()`, `.len()`, `.iter()` |
| `String` | Yes | Yes | `.push_str()`, `.push()`, `+`, `format!`, `.chars()` |
| `&str` | No (borrowed) | No | `.len()`, `.chars()`, slicing (`&s[0..3]`) |
| `HashMap<K, V>` | Yes | Yes | `.insert()`, `.get()`, `.entry().or_insert()`, `.iter()` |

## Exercise

Write a program that takes a hardcoded sentence (a `&str`), splits it into
words with `.split_whitespace()`, and uses a `HashMap<&str, i32>` with the
`entry().or_insert()` pattern to count how many times each word appears.
Print the results. Then write a second function that takes a `Vec<i32>` and
returns a new `Vec<i32>` containing only the even numbers, using a `for` loop
and `.push()`.

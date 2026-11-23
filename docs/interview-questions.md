---
title: "Rust Interview Questions and Answers (179 Questions)"
description: "179 Rust interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Rust Mastery Path."
---

# Rust Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Rust Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What tool installs and updates Rust toolchains?

**Answer:** rustup

rustup manages rustc, cargo and targets.

**Q2.** What does cargo new hello create?

**Answer:** A package with Cargo.toml and src/main.rs

cargo run builds and runs it.

**Q3.** What is the difference between cargo build and cargo check?

**Answer:** check type-checks without producing a binary, so it is faster

Use check for a quick feedback loop.

**Q4.** Where does cargo put compiled output by default?

**Answer:** target/ (debug in target/debug)

Use --release for optimised builds in target/release.

### [02 · Variables, Types &amp; Ownership Basics](level-1/02-variables-types-ownership.md)

**Q5.** What does this print?

```rust
fn main() {
    let x = 5;
    let x = x * 2;
    {
        let x = x + 1;
        println!("{x}");
    }
    println!("{x}");
    let mut y = 1;
    y += 1;
    println!("{y}");
}
```

**Answer:** 11
10
2

Shadowing creates a new binding in the current scope; the inner x ends with the block. Verified by compiling and running it.

**Q6.** What does this print?

```rust
fn main() {
    let s1 = String::from("hi");
    let s2 = s1.clone();
    let n = 5;
    let m = n;
    println!("{s1} {s2} {n} {m}");
    let t = (1, 2.5, 'c');
    println!("{} {} {}", t.0, t.1, t.2);
    println!("{}", u8::MAX);
}
```

**Answer:** hi hi 5 5
1 2.5 c
255

clone copies heap data; integers are Copy. Verified by compiling and running it.

**Q7.** What happens when you assign let b = a; where a is a String?

**Answer:** The String is moved to b and a can no longer be used

Ownership transfers on move.

**Q8.** What does integer overflow do in a debug build?

**Answer:** Panics (in release it wraps unless you use checked/wrapping methods)

Use checked_add and friends when it matters.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```rust
fn main() {
    let n = 7;
    let kind = if n % 2 == 0 { "even" } else { "odd" };
    let mut i = 0;
    let r = loop {
        i += 3;
        if i > 10 { break i; }
    };
    let mut total = 0;
    for k in (1..=4).rev() { total = total * 10 + k; }
    println!("{kind} {r} {total}");
}
```

**Answer:** odd 12 4321

if and loop are expressions; (1..=4).rev() counts down. Verified by compiling and running it.

**Q10.** What type must an if condition have?

**Answer:** bool (no implicit truthiness)

if x {} with an integer is a compile error.

**Q11.** What does break value inside loop do?

**Answer:** Ends the loop and makes the loop expression evaluate to that value

Only loop (not while/for) can return a value.

**Q12.** What does 1..5 include?

**Answer:** 1 to 4 (the end is exclusive); use 1..=5 to include 5

Ranges are half-open by default.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```rust
fn add(a: i32, b: i32) -> i32 { a + b }
fn describe(n: i32) -> &'static str {
    if n < 0 { return "neg"; }
    "non-neg"
}
fn main() {
    let v = { let t = add(2, 3); t * 2 };
    println!("{v} {} {}", describe(-1), describe(4));
}
```

**Answer:** 10 neg non-neg

A block's final expression without a semicolon is its value. Verified by compiling and running it.

**Q14.** What happens if you add a semicolon after the last expression of a function returning i32?

**Answer:** It returns () and fails to compile with a type mismatch

The semicolon turns the expression into a statement.

**Q15.** How do you return multiple values?

**Answer:** A tuple such as (i32, i32)

Destructure with let (a, b) = f();

**Q16.** What are function parameters' types required to be?

**Answer:** Annotated explicitly

Rust infers locals, not signatures.

### [05 · Structs &amp; Enums](level-1/05-structs-enums.md)

**Q17.** What does this print?

```rust
#[derive(Debug)]
enum Shape { Circle(f64), Rect { w: f64, h: f64 } }
impl Shape {
    fn area(&self) -> f64 {
        match self { Shape::Circle(r) => 3.0 * r * r, Shape::Rect { w, h } => w * h }
    }
}
struct Point { x: i32, y: i32 }
fn main() {
    let shapes = [Shape::Circle(2.0), Shape::Rect { w: 2.0, h: 4.5 }];
    for s in &shapes { println!("{:?} {}", s, s.area()); }
    let p = Point { x: 1, y: 2 };
    let q = Point { x: 5, ..p };
    println!("{} {}", q.x, q.y);
}
```

**Answer:** Circle(2.0) 12
Rect { w: 2.0, h: 4.5 } 9
5 2

f64 Display prints 12 for 12.0; struct update syntax copies the remaining fields. Verified by compiling and running it.

**Q18.** What is an enum variant with data like Circle(f64) useful for?

**Answer:** Modelling alternatives that carry different data

match must handle every variant.

**Q19.** What does #[derive(Debug)] provide?

**Answer:** An automatic implementation of the Debug trait for {:?} printing

Also Clone, PartialEq and others.

**Q20.** What are impl blocks for?

**Answer:** Defining methods and associated functions for a type

self, &amp;self or &amp;mut self choose access.

### [06 · Collections (Vec, String, HashMap)](level-1/06-collections.md)

**Q21.** What does this print?

```rust
use std::collections::HashMap;
fn main() {
    let mut v = vec![3, 1, 2];
    v.push(4);
    v.sort();
    let total: i32 = v.iter().sum();
    let mut m: HashMap<&str, i32> = HashMap::new();
    *m.entry("a").or_insert(0) += 2;
    *m.entry("a").or_insert(0) += 3;
    println!("{:?} {} {:?} {:?}", v, total, m.get("a"), m.get("z"));
}
```

**Answer:** [1, 2, 3, 4] 10 Some(5) None

entry().or_insert() inserts a default; get returns an Option. Verified by compiling and running it.

**Q22.** What does v[10] do on a Vec with 3 items?

**Answer:** Panics at runtime; v.get(10) returns None instead

Indexing is checked.

**Q23.** What is the difference between String and &amp;str?

**Answer:** String is an owned, growable buffer; &amp;str is a borrowed view into UTF-8 text

Prefer &amp;str parameters.

**Q24.** Why can't you index a String with s[0]?

**Answer:** Strings are UTF-8, so a byte index may not be a valid character boundary

Use chars() or slices at known boundaries.

### [07 · Error Handling Basics (Option, Result)](level-1/07-error-handling-basics.md)

**Q25.** What does this print?

```rust
fn parse(s: &str) -> Result<i32, std::num::ParseIntError> { s.trim().parse::<i32>() }
fn double(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n = parse(s)?;
    Ok(n * 2)
}
fn main() {
    println!("{:?}", double(" 21 "));
    println!("{}", double("x").is_err());
    let o: Option<i32> = None;
    println!("{} {}", o.unwrap_or(7), Some(3).map(|n| n + 1).unwrap());
}
```

**Answer:** Ok(42)
true
7 4

The ? operator returns early with the error. Verified by compiling and running it.

**Q26.** What does unwrap() do on an Err?

**Answer:** Panics

Use it only when failure is a bug.

**Q27.** What does ? require about the function's return type?

**Answer:** It must be a compatible Result or Option (or something that can convert the error)

main can return Result too.

**Q28.** When should you use panic!?

**Answer:** For unrecoverable bugs or violated invariants, not for expected errors

Return Result for recoverable errors.

### [08 · Pattern Matching](level-1/08-pattern-matching.md)

**Q29.** What does this print?

```rust
fn classify(n: i32) -> &'static str {
    match n {
        i32::MIN..=-1 => "neg",
        0 => "zero",
        x if x % 2 == 0 => "even",
        1 | 3 | 5 | 7 | 9 => "small odd",
        _ => "big odd",
    }
}
fn main() {
    for n in [-4, 0, 6, 7, 11] { print!("{} ", classify(n)); }
    println!();
    let pair = (3, -3);
    if let (a, b) = pair { println!("{}", a + b); }
    let opt = Some(5);
    if let Some(n @ 1..=9) = opt { println!("digit {n}"); }
}
```

**Answer:** neg zero even small odd big odd 
0
digit 5

match arms are checked top to bottom and guards add conditions. Verified by compiling and running it.

**Q30.** What does the compiler require of a match?

**Answer:** That it is exhaustive: every possible value is covered

Use _ as a catch-all.

**Q31.** When is if let better than match?

**Answer:** When you care about one pattern and ignore the rest

while let loops similarly.

**Q32.** What does the @ binding in n @ 1..=9 do?

**Answer:** Binds the matched value to n while testing the pattern

Handy for ranges.

### [09 · Modules &amp; Cargo Project Structure](level-1/09-modules-cargo.md)

**Q33.** What is a crate?

**Answer:** A compilation unit: a library or binary

A package can contain several.

**Q34.** What does pub do?

**Answer:** Makes an item visible outside its module

Items are private by default.

**Q35.** What does Cargo.lock record?

**Answer:** The exact dependency versions that were resolved

Commit it for binaries.

**Q36.** What does use std::collections::HashMap; do?

**Answer:** Brings the path into scope so you can write HashMap

It does not load anything at runtime.

### [10 · Project — CLI To-Do App](level-1/10-project-todo-cli.md)

**Q37.** What does this print?

```rust
#[derive(Debug, Clone, PartialEq)]
struct Todo { id: u32, title: String, done: bool }
fn complete(todos: &mut Vec<Todo>, id: u32) -> bool {
    match todos.iter_mut().find(|t| t.id == id) {
        Some(t) => { t.done = true; true }
        None => false,
    }
}
fn main() {
    let mut todos = vec![
        Todo { id: 1, title: "a".into(), done: false },
        Todo { id: 2, title: "b".into(), done: false },
    ];
    println!("{} {}", complete(&mut todos, 2), complete(&mut todos, 9));
    let open: Vec<_> = todos.iter().filter(|t| !t.done).map(|t| t.title.as_str()).collect();
    println!("{:?}", open);
}
```

**Answer:** true false
["a"]

iter_mut yields mutable references; filter/map/collect builds the open list. Verified by compiling and running it.

**Q38.** Why does the todo CLI take &amp;mut Vec&lt;Todo&gt; instead of Vec&lt;Todo&gt;?

**Answer:** To modify the caller's list without taking ownership

Borrow, don't move.

**Q39.** How would you persist todos between runs?

**Answer:** Serialise to a file (for example JSON with serde) and load at start

Handle missing or corrupt files.

**Q40.** Why keep the logic in functions separate from main?

**Answer:** So it can be unit-tested

main stays thin.

## Level 2 · Intermediate

### [01 · Ownership &amp; Borrowing Deep Dive](level-2/01-ownership-borrowing.md)

**Q41.** What does this print?

```rust
fn longest_len(a: &String, b: &String) -> usize { a.len().max(b.len()) }
fn append(s: &mut String) { s.push_str("!"); }
fn main() {
    let mut s = String::from("hey");
    let t = String::from("hello");
    println!("{}", longest_len(&s, &t));
    append(&mut s);
    {
        let r1 = &s;
        let r2 = &s;
        println!("{r1} {r2}");
    }
    let moved = s;
    println!("{moved} {}", t.len());
}
```

**Answer:** 5
hey! hey!
hey! 5

Many shared borrows or one mutable borrow at a time; s is moved at the end. Verified by compiling and running it.

**Q42.** What are Rust's borrowing rules?

**Answer:** Any number of shared references or exactly one mutable reference at a time, and references must not outlive the data

Enforced at compile time.

**Q43.** Why does this fail: let r; { let x = 5; r = &amp;x; } println!("{r}");?

**Answer:** x is dropped at the end of the inner block, so r would dangle

The borrow checker prevents dangling references.

**Q44.** What does Copy mean for a type?

**Answer:** Assignment duplicates the bits instead of moving, so the original stays usable

Integers, bool and char are Copy.

### [02 · Lifetimes](level-2/02-lifetimes.md)

**Q45.** What does this print?

```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str { if a.len() >= b.len() { a } else { b } }
struct Excerpt<'a> { part: &'a str }
impl<'a> Excerpt<'a> { fn first_word(&self) -> &str { self.part.split(' ').next().unwrap_or("") } }
fn main() {
    let novel = String::from("Call me Ishmael. Some years ago");
    let e = Excerpt { part: novel.split('.').next().unwrap() };
    println!("{} | {}", longest("abc", "de"), e.first_word());
}
```

**Answer:** abc | Call

The lifetime annotation ties the result to both inputs. Verified by compiling and running it.

**Q46.** What does the lifetime 'a in fn f&lt;'a&gt;(x: &amp;'a str) -&gt; &amp;'a str say?

**Answer:** The returned reference is valid for no longer than the input reference

It does not change how long data lives; it describes relationships.

**Q47.** What is 'static?

**Answer:** A lifetime lasting the whole program, such as string literals

Don't use it to silence errors.

**Q48.** When can lifetime elision omit annotations?

**Answer:** In common cases, such as one input reference, per the elision rules

Otherwise the compiler asks.

### [03 · Traits &amp; Generics](level-2/03-traits-generics.md)

**Q49.** What does this print?

```rust
trait Shape { fn area(&self) -> f64; fn name(&self) -> String { "shape".into() } }
struct Sq(f64);
struct Circle(f64);
impl Shape for Sq { fn area(&self) -> f64 { self.0 * self.0 } fn name(&self) -> String { "sq".into() } }
impl Shape for Circle { fn area(&self) -> f64 { 3.0 * self.0 * self.0 } }
fn total<T: Shape>(items: &[T]) -> f64 { items.iter().map(|s| s.area()).sum() }
fn describe(s: &dyn Shape) -> String { format!("{}:{}", s.name(), s.area()) }
fn main() {
    println!("{}", total(&[Sq(2.0), Sq(3.0)]));
    println!("{} {}", describe(&Sq(2.0)), describe(&Circle(1.0)));
}
```

**Answer:** 13
sq:4 shape:3

Default methods are used unless overridden; generics are monomorphised, dyn uses dynamic dispatch. Verified by compiling and running it.

**Q50.** What is the difference between generics with trait bounds and dyn Trait?

**Answer:** Generics are resolved at compile time (monomorphisation); dyn uses a vtable at runtime

dyn allows heterogeneous collections.

**Q51.** What does impl&lt;T: Display&gt; mean?

**Answer:** The implementation applies to any T that implements Display

A trait bound.

**Q52.** What is the orphan rule?

**Answer:** You can implement a trait for a type only if the trait or the type is local to your crate

Prevents conflicting impls.

### [04 · Closures &amp; Iterators](level-2/04-closures-iterators.md)

**Q53.** What does this print?

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6];
    let evens: Vec<i32> = v.iter().filter(|&&x| x % 2 == 0).cloned().collect();
    let sum_sq: i32 = v.iter().map(|x| x * x).sum();
    let (small, big): (Vec<i32>, Vec<i32>) = v.iter().partition(|&&x| x < 3);
    let mut offset = 10;
    let mut add = |x: i32| { offset += x; offset };
    println!("{:?} {} {:?} {:?}", evens, sum_sq, small, big);
    println!("{} {}", add(1), add(2));
    println!("{:?}", v.iter().zip("abc".chars()).collect::<Vec<_>>());
}
```

**Answer:** [2, 4, 6] 91 [1, 2] [3, 4, 5, 6]
11 13
[(1, 'a'), (2, 'b'), (3, 'c')]

Iterators are lazy until consumed; a closure that mutates captured state is FnMut. Verified by compiling and running it.

**Q54.** Why are iterator chains lazy?

**Answer:** Adaptors like map and filter do nothing until a consumer like collect or sum drives them

Zero-cost abstractions.

**Q55.** What is the difference between Fn, FnMut and FnOnce?

**Answer:** Fn only reads captures, FnMut mutates them, FnOnce consumes them

The compiler picks the least restrictive trait.

**Q56.** What does move do before a closure?

**Answer:** Forces the closure to take ownership of captured variables

Required for threads.

### [05 · Error Handling Advanced](level-2/05-error-handling-advanced.md)

**Q57.** What does this print?

```rust
use std::fmt;
#[derive(Debug)]
enum AppError { Parse(std::num::ParseIntError), Neg(i64) }
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self { AppError::Parse(e) => write!(f, "parse: {e}"), AppError::Neg(n) => write!(f, "negative: {n}") }
    }
}
impl std::error::Error for AppError {}
impl From<std::num::ParseIntError> for AppError { fn from(e: std::num::ParseIntError) -> Self { AppError::Parse(e) } }
fn read(s: &str) -> Result<i64, AppError> {
    let n: i64 = s.parse()?;
    if n < 0 { return Err(AppError::Neg(n)); }
    Ok(n)
}
fn main() {
    for s in ["5", "-3", "x"] {
        match read(s) { Ok(n) => println!("ok {n}"), Err(e) => println!("{e}") }
    }
}
```

**Answer:** ok 5
negative: -3
parse: invalid digit found in string

A From impl lets ? convert errors automatically. Verified by compiling and running it.

**Q58.** What do the thiserror and anyhow crates do?

**Answer:** thiserror derives error types for libraries; anyhow gives a flexible error type for applications

Libraries want typed errors; apps want context.

**Q59.** What does Box&lt;dyn Error&gt; allow?

**Answer:** Returning any error type from a function

At the cost of losing the concrete type.

**Q60.** What does map_err do?

**Answer:** Transforms the error of a Result

Often used with ?.

### [06 · Testing in Rust](level-2/06-testing.md)

**Q61.** What does this test module do? (Run via main for this exercise.)

```rust
fn add(a: i32, b: i32) -> i32 { a + b }
#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn adds() { assert_eq!(add(2, 3), 5); }
    #[test]
    #[should_panic(expected = "boom")]
    fn panics() { panic!("boom"); }
}
fn main() {
    println!("{}", add(2, 3));
    println!("{}", cfg!(test));
}
```

**Answer:** 5
false

Test modules are compiled only under cargo test. Verified by compiling and running it.

**Q62.** Where do integration tests live?

**Answer:** In the tests/ directory, each file a separate crate using only the public API

Unit tests sit beside the code in #[cfg(test)].

**Q63.** What does assert_eq! print on failure?

**Answer:** Both left and right values

Types need Debug and PartialEq.

**Q64.** What does cargo test -- --nocapture show?

**Answer:** println! output from passing tests

By default output of passing tests is hidden.

### [07 · Smart Pointers (Box, Rc, RefCell)](level-2/07-smart-pointers.md)

**Q65.** What does this print?

```rust
use std::rc::Rc;
use std::cell::RefCell;
fn main() {
    let shared = Rc::new(RefCell::new(vec![1]));
    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);
    a.borrow_mut().push(2);
    b.borrow_mut().push(3);
    println!("{:?} {}", shared.borrow(), Rc::strong_count(&shared));
    let bx: Box<i32> = Box::new(5);
    println!("{}", *bx + 1);
    drop(a);
    println!("{}", Rc::strong_count(&shared));
}
```

**Answer:** [1, 2, 3] 3
6
2

Rc counts owners; RefCell checks borrow rules at runtime. Verified by compiling and running it.

**Q66.** When does Rc&lt;RefCell&lt;T&gt;&gt; panic?

**Answer:** When you violate borrow rules at runtime, for example two borrow_mut at once

It moves checking from compile time to runtime.

**Q67.** What is Box&lt;T&gt; for?

**Answer:** Putting a value on the heap with single ownership, such as for recursive types or trait objects

Box&lt;dyn Trait&gt;.

**Q68.** Can Rc be sent across threads?

**Answer:** No: use Arc, which uses atomic reference counts

Rc is not Send.

### [08 · Files &amp; the Standard Library](level-2/08-files-stdlib.md)

**Q69.** What does this print?

```rust
use std::fs;
use std::io::{BufRead, BufReader, Write};
fn main() -> std::io::Result<()> {
    let dir = std::env::temp_dir().join("rs_quiz_demo.txt");
    let mut f = fs::File::create(&dir)?;
    writeln!(f, "one")?;
    writeln!(f, "two")?;
    let lines: Vec<String> = BufReader::new(fs::File::open(&dir)?).lines().collect::<Result<_, _>>()?;
    println!("{:?} {}", lines, fs::read_to_string(&dir)?.len());
    fs::remove_file(&dir)?;
    println!("{}", fs::metadata(&dir).is_err());
    Ok(())
}
```

**Answer:** ["one", "two"] 8
true

lines() strips newlines; the file holds 8 bytes. Verified by compiling and running it.

**Q70.** What does std::env::args() return?

**Answer:** An iterator over the command-line arguments (the first is the program name)

Use clap for real CLIs.

**Q71.** Why use BufReader for line-by-line reads?

**Answer:** It buffers reads to avoid a syscall per small read

Also BufWriter for writes.

**Q72.** How do you handle a missing file?

**Answer:** Match on the io::Error kind (NotFound) or propagate with ?

Give a helpful message.

### [09 · Crates &amp; Dependency Management](level-2/09-crates-dependencies.md)

**Q73.** What does cargo add serde do?

**Answer:** Adds the dependency to Cargo.toml

Version requirements follow semver.

**Q74.** What does ^1.2 mean in Cargo.toml?

**Answer:** Any version &gt;= 1.2.0 and &lt; 2.0.0

Caret is the default.

**Q75.** What do Cargo features do?

**Answer:** Enable optional parts of a crate and its dependencies

Disable default features to slim builds.

**Q76.** How can you check dependencies for known vulnerabilities?

**Answer:** cargo audit

Run in CI.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** Why model API responses as serde structs?

**Answer:** Typed deserialisation catches missing or malformed fields as errors

#[derive(Deserialize)].

**Q78.** How should the CLI react to network errors?

**Answer:** Show a clear message and a non-zero exit code

Use Result and ? with context.

**Q79.** How do you test code that calls an API?

**Answer:** Separate parsing from fetching and test parsing with sample JSON

Deterministic tests.

**Q80.** What belongs in environment variables for the CLI?

**Answer:** API keys and secrets, not source code

Never commit keys.

## Level 3 · Advanced

### [01 · Concurrency](level-3/01-concurrency.md)

**Q81.** What does this print?

```rust
use std::sync::{Arc, Mutex, mpsc};
use std::thread;
fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut hs = vec![];
    for _ in 0..4 {
        let c = Arc::clone(&counter);
        hs.push(thread::spawn(move || { for _ in 0..1000 { *c.lock().unwrap() += 1; } }));
    }
    for h in hs { h.join().unwrap(); }
    println!("{}", *counter.lock().unwrap());
    let (tx, rx) = mpsc::channel();
    for id in 0..3 { let tx = tx.clone(); thread::spawn(move || tx.send(id * 10).unwrap()); }
    drop(tx);
    let mut got: Vec<i32> = rx.iter().collect();
    got.sort();
    println!("{:?}", got);
}
```

**Answer:** 4000
[0, 10, 20]

Arc shares ownership; Mutex guards mutation; the channel closes when all senders are dropped. Verified by compiling and running it.

**Q82.** What do Send and Sync mean?

**Answer:** Send: a value can move to another thread; Sync: shared references can be used from many threads

The compiler checks them automatically.

**Q83.** What Rust guarantee does the type system provide for threads?

**Answer:** Data races are prevented at compile time (safe Rust)

Deadlocks and logic races are still possible.

**Q84.** Why is mpsc a good choice sometimes instead of shared state?

**Answer:** Message passing avoids locks by transferring ownership of data

Share memory by communicating.

### [02 · Async/Await Basics](level-3/02-async-await-basics.md)

**Q85.** What does an async fn return?

**Answer:** A Future that does nothing until it is polled (awaited)

Futures are lazy.

**Q86.** Why does async Rust need a runtime such as Tokio?

**Answer:** The standard library defines Future but not an executor to run them

Tokio provides the scheduler, I/O and timers.

**Q87.** What blocks an async executor thread?

**Answer:** Long CPU work or blocking calls without await; use spawn_blocking

Keep tasks cooperative.

**Q88.** What does tokio::spawn do?

**Answer:** Runs a future as a concurrent task

It returns a JoinHandle.

### [03 · Building CLIs with clap](level-3/03-clap-cli.md)

**Q89.** What does clap's derive API let you do?

**Answer:** Define CLI arguments as a struct with attributes and get parsing, help and errors

#[derive(Parser)].

**Q90.** What exit behaviour does clap provide for bad arguments?

**Answer:** Prints a usage error and exits with a non-zero code

--help is automatic.

**Q91.** How do you define subcommands?

**Answer:** An enum with #[derive(Subcommand)]

Like git add / commit.

**Q92.** Why validate arguments with value parsers?

**Answer:** Reject bad input early with clear messages

For example ranges.

### [04 · Databases with rusqlite](level-3/04-databases-rusqlite.md)

**Q93.** What does rusqlite provide?

**Answer:** A safe Rust interface to SQLite

Use parameters, not string formatting.

**Q94.** How do you avoid SQL injection with rusqlite?

**Answer:** Use parameterised queries (params!) not format!

Bind values.

**Q95.** What does a transaction give you?

**Answer:** All-or-nothing changes across several statements

commit or roll back.

**Q96.** What index helps WHERE email = ?

**Answer:** An index on email

Indexes trade write cost for read speed.

### [05 · Web Services with axum](level-3/05-web-services-axum.md)

**Q97.** What does an Axum handler return?

**Answer:** Something that implements IntoResponse, such as Json&lt;T&gt; or a status code tuple

Extractors parse the request.

**Q98.** What is an extractor in Axum?

**Answer:** A handler argument type (Path, Query, Json, State) that pulls data from the request

Order matters for body extractors.

**Q99.** How do you share a database pool across handlers?

**Answer:** With State&lt;AppState&gt; holding a cloneable handle

Arc-based.

**Q100.** What status code means 'not found'?

**Answer:** 404

Use proper codes.

### [06 · Unsafe Rust](level-3/06-unsafe-rust.md)

**Q101.** What does unsafe allow?

**Answer:** Operations the compiler can't verify, such as dereferencing raw pointers and calling unsafe functions

Everything else still applies.

**Q102.** What is your responsibility inside an unsafe block?

**Answer:** Upholding the invariants the compiler can't check

Document them in a SAFETY comment.

**Q103.** What should you do to limit unsafe code?

**Answer:** Wrap it in a small, tested, safe abstraction

Minimise the surface.

**Q104.** Which tool detects undefined behaviour in unsafe code?

**Answer:** Miri

cargo +nightly miri test.

### [07 · Design Patterns in Rust](level-3/07-design-patterns.md)

**Q105.** What is the builder pattern for?

**Answer:** Constructing complex values step by step with optional parts

Method chaining returning self.

**Q106.** What does the newtype pattern give you?

**Answer:** A distinct type wrapping another for type safety (e.g. UserId(u64))

Prevents mixing IDs.

**Q107.** What is RAII in Rust?

**Answer:** Resources are released when the owning value is dropped (Drop)

Guards unlock on drop.

**Q108.** What is the typestate pattern?

**Answer:** Encoding states in types so invalid transitions don't compile

Compile-time protocol checks.

### [08 · Testing &amp; Benchmarking](level-3/08-testing-benchmarking.md)

**Q109.** What does cargo bench or criterion measure?

**Answer:** Performance of code under repeated runs

Use release mode.

**Q110.** What does #[ignore] do on a test?

**Answer:** Skips it unless run with --ignored

For slow tests.

**Q111.** What is property-based testing?

**Answer:** Generating many random inputs to check that properties always hold (proptest)

Shrinks failures.

**Q112.** Why run tests with cargo test --release sometimes?

**Answer:** Optimised builds can expose overflow and timing differences

Debug panics on overflow; release wraps.

### [09 · Generics &amp; Trait Objects](level-3/09-generics-trait-objects.md)

**Q113.** What is a trait object?

**Answer:** A value behind a pointer used through dyn Trait with dynamic dispatch

Box&lt;dyn Trait&gt;.

**Q114.** What is object safety?

**Answer:** Only certain traits can be made into dyn (no generic methods or Self returns)

The compiler tells you.

**Q115.** When would you pick generics over dyn?

**Answer:** For performance and when types are known at compile time

Monomorphisation can bloat code.

**Q116.** What does impl Trait in argument position mean?

**Answer:** Syntactic sugar for a generic parameter with that bound

fn f(x: impl Display).

### [10 · Project: REST API with SQLite](level-3/10-project-rest-api-sqlite.md)

**Q117.** In the REST API project, where should SQL live?

**Answer:** In a repository/data layer separate from the HTTP handlers

Easier to test.

**Q118.** Why map database errors to HTTP statuses explicitly?

**Answer:** So clients get meaningful responses instead of generic 500s

For example 404 and 409.

**Q119.** What does a 201 response include?

**Answer:** The created resource and often a Location header

POST creates.

**Q120.** How do you test handlers without a network?

**Answer:** Call the router with a request in tests (tower::ServiceExt::oneshot)

Fast, in-process.

## Level 4 · Master

### [01 · Advanced Async Patterns](level-4/01-advanced-async-patterns.md)

**Q121.** What does tokio::select! do?

**Answer:** Waits on several futures and acts on the first that completes

Cancels the others.

**Q122.** What is backpressure?

**Answer:** Slowing producers when consumers can't keep up, for example with bounded channels

Avoid unbounded queues.

**Q123.** What is cancellation safety?

**Answer:** Dropping a future mid-way must not lose data or leave inconsistent state

Check the docs of what you select on.

**Q124.** How do you limit concurrency?

**Answer:** A Semaphore or buffer_unordered(n)

Protect downstreams.

### [02 · Performance Optimization](level-4/02-performance-optimization.md)

**Q125.** What is the first step in optimising Rust code?

**Answer:** Measure with a profiler or benchmarks in release mode

Optimise hot paths only.

**Q126.** Why prefer iterators over indexing in loops?

**Answer:** They avoid bounds checks and often optimise as well or better

Zero-cost abstractions.

**Q127.** What does Vec::with_capacity do?

**Answer:** Preallocates space to avoid repeated reallocation

Use when the size is known.

**Q128.** What does lto = true in release profile do?

**Answer:** Enables link-time optimisation across crates for smaller/faster code at longer build time

Also codegen-units = 1.

### [03 · Macros](level-4/03-macros.md)

**Q129.** What is a declarative macro?

**Answer:** macro_rules! pattern-based code generation

Matches token patterns.

**Q130.** What are procedural macros?

**Answer:** Rust functions that transform token streams, used for derives and attributes

For example #[derive(Serialize)].

**Q131.** When should you prefer a function over a macro?

**Answer:** Almost always; use macros only when you need syntax or code generation a function cannot do

Macros are harder to debug.

**Q132.** How can you inspect macro expansion?

**Answer:** cargo expand

Shows generated code.

### [04 · Production Services](level-4/04-production-services.md)

**Q133.** What do tracing and log crates provide?

**Answer:** Structured, leveled logging and spans for diagnostics

Instrument async code with spans.

**Q134.** What does graceful shutdown mean for a service?

**Answer:** Stop accepting new work, finish in-flight requests, then exit on a signal

Handle SIGTERM.

**Q135.** Why add timeouts to outgoing calls?

**Answer:** A hung dependency would otherwise tie up your tasks indefinitely

Also retries with backoff.

**Q136.** What should /health report?

**Answer:** Whether the service and its critical dependencies are working

Separate liveness from readiness.

### [05 · Security &amp; Memory Safety](level-4/05-security-memory-safety.md)

**Q137.** What kinds of bugs does safe Rust prevent?

**Answer:** Use-after-free, double free, buffer overflows and data races

Logic and security flaws remain.

**Q138.** Why audit dependencies?

**Answer:** Supply-chain vulnerabilities live in crates you pull in

cargo audit and cargo deny.

**Q139.** What can still go wrong in unsafe code?

**Answer:** Undefined behaviour from violated invariants

Review carefully.

**Q140.** Why zeroize secrets in memory?

**Answer:** To reduce exposure of keys after use

The zeroize crate.

### [06 · Testing at Scale &amp; CI](level-4/06-testing-at-scale-ci.md)

**Q141.** What does cargo clippy do?

**Answer:** Lints for common mistakes and non-idiomatic code

Run in CI with -D warnings.

**Q142.** What does cargo fmt --check do in CI?

**Answer:** Fails if code isn't formatted

Consistency.

**Q143.** How can you speed up CI builds?

**Answer:** Cache target and registry (sccache or Swatinem/rust-cache)

Compile times add up.

**Q144.** Why test on multiple Rust versions?

**Answer:** To verify your minimum supported version (MSRV) and the latest

Declare rust-version.

### [07 · Deployment with Docker](level-4/07-deployment-docker.md)

**Q145.** Why use a multi-stage Docker build for Rust?

**Answer:** Compile in a big image and copy only the binary into a small runtime image

Smaller attack surface.

**Q146.** What does cargo-chef help with?

**Answer:** Caching dependency compilation in Docker layers

Faster rebuilds.

**Q147.** Why run the container as non-root?

**Answer:** Limits damage if compromised

Least privilege.

**Q148.** What is a static musl build for?

**Answer:** A mostly self-contained binary that runs in minimal images

Watch for DNS and OpenSSL caveats.

### [08 · FFI Basics](level-4/08-ffi-basics.md)

**Q149.** What does extern "C" do?

**Answer:** Declares functions using the C ABI for interop

Pair with #[no_mangle] to export.

**Q150.** Why is calling C from Rust unsafe?

**Answer:** The compiler can't verify the foreign code's guarantees

Wrap in a safe API.

**Q151.** What does bindgen do?

**Answer:** Generates Rust bindings from C headers

Common in -sys crates.

**Q152.** What must you ensure for strings crossing FFI?

**Answer:** Null termination and ownership rules (CString/CStr)

Document who frees memory.

### [09 · Workspace Organization](level-4/09-workspace-organization.md)

**Q153.** What is a Cargo workspace?

**Answer:** Several crates sharing one Cargo.lock and target directory

Defined in the root Cargo.toml.

**Q154.** Why split a project into a workspace?

**Answer:** Clear boundaries, faster incremental builds and reuse

For example core, api and cli crates.

**Q155.** How can members share dependency versions?

**Answer:** [workspace.dependencies] and dep.workspace = true

Keeps versions aligned.

**Q156.** What should a core library crate avoid depending on?

**Answer:** Application or framework specifics such as HTTP servers

Keep it portable and testable.

### [10 · Capstone: URL Shortener Service](level-4/10-capstone-project.md)

**Q157.** What is the capstone's aim?

**Answer:** A production-style Rust service with tests, CI, observability, security and a container image

Show readiness.

**Q158.** What belongs in the README?

**Answer:** How to build, run, test and deploy, plus design decisions

Reviewers start there.

**Q159.** What proves the service is production-ready?

**Answer:** Passing CI, clippy and fmt clean, load and failure testing, and a documented runbook

Evidence over claims.

**Q160.** Why include metrics and structured logs?

**Answer:** So you can see failures and latency in production

Observability.

## Mixed questions across all levels

**Q161.** What does this print?

```rust
fn first_word(s: &str) -> &str { s.split(' ').next().unwrap_or("") }
fn main() {
    let owned = String::from("hello world");
    let w = first_word(&owned);
    println!("{w} {}", owned.len());
    let v: Vec<i32> = Vec::new();
    println!("{:?} {}", v.first(), v.is_empty());
}
```

**Answer:** hello 11
None true

Borrowed slices share the owner's data; first() on an empty Vec is None. Verified by compiling and running it.

**Q162.** What does this print?

```rust
use std::collections::HashMap;
fn main() {
    let words = ["a", "b", "a", "c", "a", "b"];
    let mut m: HashMap<&str, usize> = HashMap::new();
    for w in words { *m.entry(w).or_default() += 1; }
    let mut pairs: Vec<_> = m.into_iter().collect();
    pairs.sort_by(|x, y| y.1.cmp(&x.1).then(x.0.cmp(y.0)));
    println!("{:?}", pairs);
}
```

**Answer:** [("a", 3), ("b", 2), ("c", 1)]

HashMap iteration order is unspecified, so sort before printing. Verified by compiling and running it.

**Q163.** What does this print?

```rust
fn apply<F: Fn(i32) -> i32>(f: F, n: i32) -> i32 { f(f(n)) }
fn make_adder(k: i32) -> impl Fn(i32) -> i32 { move |x| x + k }
fn boxed(k: i32) -> Box<dyn Fn(i32) -> i32> { Box::new(move |x| x * k) }
fn main() {
    println!("{} {} {}", apply(make_adder(3), 1), boxed(4)(5), apply(|x| x * x, 3));
}
```

**Answer:** 7 20 81

impl Fn returns one concrete closure type; Box&lt;dyn Fn&gt; allows different closures at runtime. Verified by compiling and running it.

**Q164.** What does this print?

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
enum Level { Low = 1, Mid, High }
fn main() {
    let mut v = vec![Level::High, Level::Low, Level::Mid];
    v.sort();
    println!("{:?} {} {}", v, Level::Mid as i32, Level::Low < Level::High);
}
```

**Answer:** [Low, Mid, High] 2 true

Derived Ord follows declaration order; discriminants continue counting. Verified by compiling and running it.

**Q165.** What does this print?

```rust
fn main() {
    let data = vec![String::from("a"), String::from("bb")];
    let lens: Vec<usize> = data.iter().map(|s| s.len()).collect();
    let joined = data.join("-");
    let moved = data;
    println!("{:?} {} {}", lens, joined, moved.len());
    let r: Result<u8, _> = "300".parse::<u8>();
    println!("{}", r.is_err());
}
```

**Answer:** [1, 2] a-bb 2
true

Borrowing for iter() ends before the move; 300 does not fit in u8. Verified by compiling and running it.

**Q166.** What does the borrow checker prevent in this code: let r; { let x = 1; r = &amp;x; } use(r)?

**Answer:** A dangling reference to x after its scope ends

References cannot outlive their referent.

**Q167.** A function takes &amp;mut Vec&lt;T&gt; while you hold an immutable reference to the vec. Why does it fail to compile?

**Answer:** A mutable borrow cannot coexist with a shared borrow

Aliasing XOR mutability.

**Q168.** When should a library return Result instead of panicking?

**Answer:** For any failure a caller might reasonably handle

Panic for bugs and broken invariants.

**Q169.** What does the ? operator do on an Err?

**Answer:** Returns early from the function, converting the error with From

It works on Option too.

**Q170.** What is the difference between Rc&lt;T&gt; and Arc&lt;T&gt;?

**Answer:** Arc uses atomic counts and can be shared across threads; Rc cannot

Rc is cheaper in single-threaded code.

**Q171.** Why wrap shared mutable data in Arc&lt;Mutex&lt;T&gt;&gt;?

**Answer:** Arc shares ownership across threads and Mutex ensures exclusive mutable access

Lock scope should be short.

**Q172.** What does an async fn do when called without .await?

**Answer:** Creates a Future that does nothing until polled

Futures are lazy.

**Q173.** What is the safest way to run blocking work inside Tokio?

**Answer:** tokio::task::spawn_blocking

Keep async threads free.

**Q174.** What does #[derive(Deserialize)] with serde give you?

**Answer:** Generated code to parse formats such as JSON into your struct

Errors for missing or wrong-typed fields.

**Q175.** What does cargo clippy -- -D warnings do in CI?

**Answer:** Fails the build on lint warnings

Keeps quality steady.

**Q176.** What is the purpose of a SAFETY comment on an unsafe block?

**Answer:** To document why the invariants hold

Reviewers rely on it.

**Q177.** Why prefer a multi-stage Docker build?

**Answer:** Ship only the compiled binary in a small image

Smaller attack surface.

**Q178.** What does a Cargo workspace share between crates?

**Answer:** The Cargo.lock and target directory

Consistent versions and faster builds.

**Q179.** Which evidence best shows a Rust service is ready for production?

**Answer:** Tests, clippy and fmt in CI, security audit, logs and metrics and a runbook

Operational readiness.

## Practical coding questions

### A ledger with typed errors

Implement apply for a small ledger: parse each line, update balances in a BTreeMap and report failures with a custom error enum, a Display impl and a From conversion so the ? operator works. Valid lines must still be applied when others fail.

Starter code:

```
use std::collections::BTreeMap;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug, PartialEq)]
enum LedgerError {
    BadLine(String),
    BadAmount(ParseIntError),
    Overdraft { account: String, balance: i64 },
}

impl fmt::Display for LedgerError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            LedgerError::BadLine(l) => write!(f, "bad line: {l}"),
            LedgerError::BadAmount(e) => write!(f, "bad amount: {e}"),
            LedgerError::Overdraft { account, balance } => write!(f, "overdraft on {account}: {balance}"),
        }
    }
}

impl From<ParseIntError> for LedgerError {
    fn from(e: ParseIntError) -> Self { LedgerError::BadAmount(e) }
}

// TODO: fn apply(balances: &mut BTreeMap<String, i64>, line: &str) -> Result<(), LedgerError>
//  - a line is "<account> <deposit|withdraw> <amount>"
//  - wrong number of parts or an unknown verb -> BadLine(line)
//  - amount parse failure -> BadAmount via the ? operator and the From impl
//  - a withdraw that would make the balance negative -> Overdraft { account, balance } (balance before the attempt), and the balance is unchanged

fn main() {
    let lines = ["ana deposit 100", "bo deposit 50", "ana withdraw 30", "bo withdraw 80", "cy deposit x1", "ana transfer 5", "ana deposit", "bo withdraw 50"];
    let mut balances = BTreeMap::new();
    let mut errors = 0;
    for l in lines {
        if let Err(e) = apply(&mut balances, l) {
            errors += 1;
            println!("error: {e}");
        }
    }
    for (name, b) in &balances { println!("{name}: {b}"); }
    println!("errors: {errors}");
}
```

Expected output:

```text
error: overdraft on bo: 50
error: bad amount: invalid digit found in string
error: bad line: ana transfer 5
error: bad line: ana deposit
ana: 70
bo: 0
errors: 4
```

??? success "Show a solution"

    ```
    use std::collections::BTreeMap;
    use std::fmt;
    use std::num::ParseIntError;

    #[derive(Debug, PartialEq)]
    enum LedgerError {
        BadLine(String),
        BadAmount(ParseIntError),
        Overdraft { account: String, balance: i64 },
    }

    impl fmt::Display for LedgerError {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            match self {
                LedgerError::BadLine(l) => write!(f, "bad line: {l}"),
                LedgerError::BadAmount(e) => write!(f, "bad amount: {e}"),
                LedgerError::Overdraft { account, balance } => write!(f, "overdraft on {account}: {balance}"),
            }
        }
    }

    impl From<ParseIntError> for LedgerError {
        fn from(e: ParseIntError) -> Self { LedgerError::BadAmount(e) }
    }

    fn apply(balances: &mut BTreeMap<String, i64>, line: &str) -> Result<(), LedgerError> {
        let parts: Vec<&str> = line.split_whitespace().collect();
        if parts.len() != 3 { return Err(LedgerError::BadLine(line.to_string())); }
        let amount: i64 = parts[2].parse()?;
        let delta = match parts[1] {
            "deposit" => amount,
            "withdraw" => -amount,
            _ => return Err(LedgerError::BadLine(line.to_string())),
        };
        let bal = balances.entry(parts[0].to_string()).or_insert(0);
        if *bal + delta < 0 {
            return Err(LedgerError::Overdraft { account: parts[0].to_string(), balance: *bal });
        }
        *bal += delta;
        Ok(())
    }

    fn main() {
        let lines = ["ana deposit 100", "bo deposit 50", "ana withdraw 30", "bo withdraw 80", "cy deposit x1", "ana transfer 5", "ana deposit", "bo withdraw 50"];
        let mut balances = BTreeMap::new();
        let mut errors = 0;
        for l in lines {
            if let Err(e) = apply(&mut balances, l) {
                errors += 1;
                println!("error: {e}");
            }
        }
        for (name, b) in &balances { println!("{name}: {b}"); }
        println!("errors: {errors}");
    }
    ```

    The match on the verb gives a signed delta, entry().or_insert(0) creates accounts on demand and the overdraft check happens before the balance changes. The From&lt;ParseIntError&gt; impl is what lets ? convert the parse error. Verified by compiling and running with Rust 1.97.

### Count primes with worker threads

Share a counter in an Arc&lt;Mutex&lt;u32&gt;&gt; so worker threads claim chunks of 1000 numbers, count primes in each chunk and send results over an mpsc channel. The totals must match a single-worker run.

Starter code:

```
use std::sync::mpsc;
use std::sync::{Arc, Mutex};
use std::thread;

fn is_prime(n: u32) -> bool {
    if n < 2 { return false; }
    let mut d = 2;
    while d * d <= n {
        if n % d == 0 { return false; }
        d += 1;
    }
    true
}

// TODO: fn count_primes(limit: u32, workers: u32) -> (u32, Vec<(u32, u32)>)
//  - workers threads repeatedly claim the next chunk of 1000 numbers from a shared Arc<Mutex<u32>>
//  - each worker counts primes in its chunk and sends (chunk_start, count) over an mpsc channel
//  - return the total and the chunk results sorted by chunk start

fn main() {
    let (total, parts) = count_primes(10_000, 4);
    println!("primes below 10000: {total}");
    println!("chunks: {}", parts.len());
    println!("first three: {:?}", &parts[..3]);
    let (t1, _) = count_primes(10_000, 1);
    println!("same with one worker: {}", t1 == total);
}
```

Expected output:

```text
primes below 10000: 1229
chunks: 10
first three: [(0, 168), (1000, 135), (2000, 127)]
same with one worker: true
```

??? success "Show a solution"

    ```
    use std::sync::mpsc;
    use std::sync::{Arc, Mutex};
    use std::thread;

    fn is_prime(n: u32) -> bool {
        if n < 2 { return false; }
        let mut d = 2;
        while d * d <= n {
            if n % d == 0 { return false; }
            d += 1;
        }
        true
    }

    fn count_primes(limit: u32, workers: u32) -> (u32, Vec<(u32, u32)>) {
        let next = Arc::new(Mutex::new(0u32));
        let chunk = 1000;
        let (tx, rx) = mpsc::channel();
        let mut handles = Vec::new();
        for id in 0..workers {
            let next = Arc::clone(&next);
            let tx = tx.clone();
            handles.push(thread::spawn(move || {
                loop {
                    let start = {
                        let mut g = next.lock().unwrap();
                        let s = *g;
                        *g += chunk;
                        s
                    };
                    if start >= limit { break; }
                    let end = (start + chunk).min(limit);
                    let c = (start..end).filter(|&n| is_prime(n)).count() as u32;
                    tx.send((start, c)).unwrap();
                }
                id
            }));
        }
        drop(tx);
        let mut parts: Vec<(u32, u32)> = rx.iter().collect();
        for h in handles { h.join().unwrap(); }
        parts.sort();
        (parts.iter().map(|p| p.1).sum(), parts)
    }

    fn main() {
        let (total, parts) = count_primes(10_000, 4);
        println!("primes below 10000: {total}");
        println!("chunks: {}", parts.len());
        println!("first three: {:?}", &parts[..3]);
        let (t1, _) = count_primes(10_000, 1);
        println!("same with one worker: {}", t1 == total);
    }
    ```

    The lock is held only while claiming the next chunk, the channel closes when every sender is dropped (the original tx is dropped explicitly) and sorting the results makes the output deterministic even though threads finish in any order. 1229 is the number of primes below 10,000. Verified by compiling and running with Rust 1.97.

---

More free practice: the [Rust Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).

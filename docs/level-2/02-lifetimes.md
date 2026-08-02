# 02 · Lifetimes

The previous module showed that a reference can never outlive the data it
points to — the compiler rejects dangling references before the program
runs. **Lifetimes** are the mechanism that makes this checking possible:
they're a way of naming "how long a reference is valid for" so the compiler
can verify your code respects it, especially once references start flowing
through function signatures and structs instead of staying inside one block.

## Lifetimes are not runtime durations

This is the biggest misconception to clear up first: a lifetime annotation
doesn't change how long anything lives, and it has zero runtime cost. It's
purely a piece of information for the *compiler* — a label that lets it
match up "this reference" with "the scope it's allowed to be valid in," the
same way a type annotation doesn't change a value's bits, just what the
compiler is allowed to check about it.

## The problem lifetimes solve

```rust
fn longest(x: &str, y: &str) -> &str {   // ERROR: missing lifetime specifier
    if x.len() > y.len() { x } else { y }
}
```

```text
error[E0106]: missing lifetime specifier
help: this function's return type contains a borrowed value, but the
signature does not say whether it is borrowed from `x` or `y`
```

The function body is fine — the problem is the *signature*. The return
value is a reference, but the compiler has no way to know, just from
`&str` and `&str`, whether the returned reference's validity is tied to `x`
or to `y`. Without that information it can't check call sites for safety, so
it refuses to guess.

## Fixing it with a lifetime parameter

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("hello");
    let s2 = String::from("world!!");

    let result = longest(&s1, &s2);
    println!("longest: {result}");   // longest: world!!
}
```

```text
longest: world!!
```

`'a` (read "tick-a") is a generic lifetime parameter, declared in angle
brackets just like a type parameter. `x: &'a str, y: &'a str, -> &'a str`
says: "the returned reference is valid for exactly as long as *both* `x` and
`y` are valid" — in practice, the compiler picks the smaller of the two
input lifetimes as the constraint. This isn't extra restriction the
annotation *adds*; it's documenting a constraint that was always true of the
function body, so the compiler can enforce it at every call site.

## Lifetimes prevent the caller from misusing the result

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long string is long");
    let result;
    {
        let s2 = String::from("short");
        result = longest(s1.as_str(), s2.as_str());
        println!("longest: {result}");   // fine, s2 is still alive here
    }
    // println!("{result}");
    // ERROR: `s2` does not live long enough -- `result` might still
    // reference it, but s2 was dropped at the end of the inner block
}
```

```text
longest: long string is long
```

This is exactly the payoff: the compiler used the `'a` constraint to figure
out that `result` could be borrowed from `s2`, and since `s2` dies at the
end of the inner block, using `result` afterward is rejected — a would-be
dangling reference caught before the program ever runs.

## Lifetime elision: why you don't always write `'a`

Most functions that take and return references *don't* need explicit
lifetime annotations — the compiler applies three elision rules
automatically:

1. Each elided input reference gets its own lifetime parameter.
2. If there's exactly one input lifetime, it's assigned to all elided output
   lifetimes.
3. If one parameter is `&self` or `&mut self`, its lifetime is assigned to
   all elided output lifetimes (methods almost always tie the return value
   to `self`).

```rust
// No annotations needed -- rule 2 applies: one input reference,
// so the output borrows from it.
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(pos) => &s[..pos],
        None => s,
    }
}

fn main() {
    let sentence = String::from("hello there world");
    println!("{}", first_word(&sentence));   // hello
}
```

The `longest` function from earlier needed explicit `'a` precisely because
it breaks rule 2 and 3: there are *two* input references, and neither is
`&self`, so the compiler has no default to fall back on — you have to state
the relationship yourself.

## Lifetimes in structs

A struct that holds a reference must declare a lifetime parameter, because
the struct can't outlive the data its field borrows.

```rust
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    fn announce(&self) {
        println!("Excerpt: {}", self.text);
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("no sentence found");

    let excerpt = Excerpt { text: first_sentence };
    excerpt.announce();   // Excerpt: Call me Ishmael
}
```

```text
Excerpt: Call me Ishmael
```

`Excerpt<'a>` can never outlive `novel`, because `text` borrows from it — the
compiler enforces that at every use of `excerpt`, the same way it enforces
plain reference lifetimes.

## The `'static` lifetime

`'static` means "valid for the entire duration of the program." String
literals are `'static` because they're baked directly into the compiled
binary:

```rust
fn main() {
    let s: &'static str = "I live for the whole program";
    println!("{s}");
}
```

`'static` shows up a lot in error messages from beginners trying to satisfy
the borrow checker by slapping `'static` on everything — resist that urge.
It's the right answer only when the data genuinely lives forever (string
literals, values leaked intentionally, or globals); using it to paper over a
lifetime error usually just moves the error somewhere else, or forces you to
`.clone()` data that didn't need cloning.

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `&'a str` | A reference valid for lifetime `'a` |
| `fn f<'a>(x: &'a str) -> &'a str` | Output lifetime tied to input `x` |
| `struct S<'a> { field: &'a str }` | Struct can't outlive the data `field` borrows |
| `'static` | Valid for the whole program (string literals, leaked/global data) |
| Elision rule 1 | Each elided input reference gets its own lifetime |
| Elision rule 2 | One input lifetime → applied to all elided outputs |
| Elision rule 3 | `&self`/`&mut self` lifetime → applied to all elided outputs |

## Exercise

Write a struct `struct Highlight<'a> { word: &'a str, context: &'a str }`
with a method `fn show(&self)` that prints `"{word}" found in: {context}`.
Then write a free function
`fn find_highlight<'a>(context: &'a str, word: &'a str) -> Option<Highlight<'a>>`
that returns `Some(Highlight { word, context })` if `context.contains(word)`
is true, otherwise `None`. In `main`, call it with a sentence and a word that
appears in it, print the result with `.show()`, and add a comment explaining
which elision rule (if any) would have let you omit the lifetimes if this
were a method on `context` instead of a free function.

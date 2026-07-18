# 05 · Structs & Enums

Structs group related data together under one type. Enums represent a value
that can be one of several distinct variants — Rust's enums are far more
powerful than in most languages, since each variant can carry its own data.

## Defining and using structs

```rust
struct Person {
    name: String,
    age: u32,
}

fn main() {
    let ada = Person {
        name: String::from("Ada"),
        age: 30,
    };

    println!("{} is {} years old", ada.name, ada.age);
}
```

```text
Ada is 30 years old
```

## Structs are immutable by default

```rust
struct Person {
    name: String,
    age: u32,
}

fn main() {
    let mut ada = Person {
        name: String::from("Ada"),
        age: 30,
    };

    ada.age += 1;   // requires `mut` on the binding
    println!("{}", ada.age); // 31
}
```

## Methods with `impl`

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    // Associated function (no `self`) -- acts like a constructor
    fn new(width: f64, height: f64) -> Rectangle {
        Rectangle { width, height }
    }

    // Method (takes `&self`) -- operates on an existing instance
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

fn main() {
    let rect = Rectangle::new(3.0, 4.0);
    println!("{}", rect.area()); // 12
}
```

## Enums — a value that's one of several variants

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn describe(dir: &Direction) -> &str {
    match dir {
        Direction::North => "heading north",
        Direction::South => "heading south",
        Direction::East => "heading east",
        Direction::West => "heading west",
    }
}

fn main() {
    let d = Direction::North;
    println!("{}", describe(&d)); // heading north
}
```

## Enums that carry data (this is what makes Rust enums special)

```rust
enum Shape {
    Circle(f64),           // radius
    Rectangle(f64, f64),   // width, height
    Triangle(f64, f64),    // base, height
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(radius) => std::f64::consts::PI * radius * radius,
        Shape::Rectangle(width, height) => width * height,
        Shape::Triangle(base, height) => 0.5 * base * height,
    }
}

fn main() {
    let shapes = vec![
        Shape::Circle(2.0),
        Shape::Rectangle(3.0, 4.0),
        Shape::Triangle(6.0, 2.0),
    ];

    for shape in &shapes {
        println!("{:.2}", area(shape));
    }
}
```

```text
12.57
12.00
6.00
```

Each variant of `Shape` carries exactly the data it needs — a `Circle` only
stores a radius, a `Rectangle` stores two dimensions. `match` forces you to
handle every variant, so adding a new shape later means the compiler flags
every `match` that needs updating.

## Deriving common traits

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();

    println!("{:?}", p1);        // Point { x: 1, y: 2 }
    println!("{}", p1 == p2);    // true
}
```

`#[derive(...)]` auto-generates common trait implementations: `Debug` enables
`{:?}` printing, `Clone` enables `.clone()`, `PartialEq` enables `==`. Adding
these attributes is far more common than hand-writing the trait
implementations yourself.

## Cheat sheet

| Concept | Syntax |
|---------|--------|
| Define a struct | `struct Name { field: Type, ... }` |
| Instantiate | `Name { field: value, ... }` |
| Method | `impl Name { fn method(&self) -> T { ... } }` |
| Associated function | `impl Name { fn new(...) -> Name { ... } }` |
| Define an enum | `enum Name { VariantA, VariantB(Type), ... }` |
| Match on an enum | `match value { Name::VariantA => ..., ... }` |
| Auto-derive traits | `#[derive(Debug, Clone, PartialEq)]` |

## Exercise

Define an enum `Command` with variants `Add(i32, i32)`, `Subtract(i32, i32)`,
and `Quit`. Write a function `execute(cmd: &Command) -> Option<i32>` that
returns `Some(result)` for `Add`/`Subtract`, and `None` for `Quit`. Then
define a `struct Calculator` with a `history: Vec<i32>` field and a method
`run(&mut self, cmd: Command)` that calls `execute`, and if it returns
`Some(value)`, pushes `value` onto `history`.

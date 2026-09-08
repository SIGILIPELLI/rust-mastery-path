# 10 · Project — CLI To-Do App

A small end-to-end project combining everything from Level 1: structs,
enums, ownership, collections, error handling, pattern matching, and modules.

## What you'll build

A command-line to-do list that:

- Adds tasks
- Lists tasks (with done/pending status)
- Marks tasks done
- Deletes tasks
- Persists everything to a JSON file between runs

## Project layout

```text
todo_app/
    Cargo.toml
    src/
        main.rs
        storage.rs
```

```toml
# Cargo.toml
[package]
name = "todo_app"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

## src/storage.rs — persistence layer

```rust
// src/storage.rs
use serde::{Deserialize, Serialize};
use std::fs;

#[derive(Serialize, Deserialize, Debug)]
pub struct Task {
    pub description: String,
    pub done: bool,
}

const DB_PATH: &str = "tasks.json";

pub fn load_tasks() -> Vec<Task> {
    match fs::read_to_string(DB_PATH) {
        Ok(contents) => serde_json::from_str(&contents).unwrap_or_default(),
        Err(_) => Vec::new(),   // file doesn't exist yet -- start empty
    }
}

pub fn save_tasks(tasks: &[Task]) {
    let json = serde_json::to_string_pretty(tasks).expect("failed to serialize tasks");
    fs::write(DB_PATH, json).expect("failed to write tasks file");
}
```

## src/main.rs — CLI logic

```rust
// src/main.rs
mod storage;

use std::env;
use storage::{load_tasks, save_tasks, Task};

fn add_task(tasks: &mut Vec<Task>, description: String) {
    tasks.push(Task {
        description: description.clone(),
        done: false,
    });
    println!("Added: {}", description);
}

fn list_tasks(tasks: &[Task]) {
    if tasks.is_empty() {
        println!("No tasks yet.");
        return;
    }

    for (i, task) in tasks.iter().enumerate() {
        let status = if task.done { "x" } else { " " };
        println!("[{}] {}. {}", status, i + 1, task.description);
    }
}

fn complete_task(tasks: &mut [Task], index: usize) {
    match tasks.get_mut(index - 1) {
        Some(task) => {
            task.done = true;
            println!("Marked task {} done.", index);
        }
        None => println!("No task with number {}", index),
    }
}

fn delete_task(tasks: &mut Vec<Task>, index: usize) {
    if index == 0 || index > tasks.len() {
        println!("No task with number {}", index);
        return;
    }
    let removed = tasks.remove(index - 1);
    println!("Deleted: {}", removed.description);
}

fn main() {
    let mut tasks = load_tasks();
    let args: Vec<String> = env::args().skip(1).collect();

    if args.is_empty() {
        println!("Usage: todo [add <text> | list | done <n> | delete <n>]");
        return;
    }

    match args[0].as_str() {
        "add" if args.len() > 1 => {
            add_task(&mut tasks, args[1..].join(" "));
            save_tasks(&tasks);
        }
        "list" => list_tasks(&tasks),
        "done" if args.len() > 1 => {
            if let Ok(n) = args[1].parse::<usize>() {
                complete_task(&mut tasks, n);
                save_tasks(&tasks);
            }
        }
        "delete" if args.len() > 1 => {
            if let Ok(n) = args[1].parse::<usize>() {
                delete_task(&mut tasks, n);
                save_tasks(&tasks);
            }
        }
        _ => println!("Unknown command."),
    }
}
```

## Running it

```bash
cargo run -- add "Write Level 1 exercises"
cargo run -- add "Review Level 2 outline"
cargo run -- list
# [ ] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline

cargo run -- done 1
cargo run -- list
# [x] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline
```

## How It Actually Works

`std::env::args()` gives you an iterator over the process's argv, already
decoded from the OS into Rust `String`s — on Unix that means the kernel
handed your process a raw array of C-string pointers at `exec`, and the Rust
runtime's startup code (`std::rt`) converts each one into an owned,
UTF-8-validated `String` before `main` ever runs, panicking early if a
argument isn't valid UTF-8 rather than letting invalid bytes propagate
silently. Every `Task` in your `Vec<Task>` lives in one contiguous heap
allocation the vector owns; `tasks.iter_mut().find(...)` walks that buffer
by reference rather than copying tasks out, so marking one done mutates it
in place with no extra allocation.

The `match command.as_str() { "add" => ..., "list" => ..., _ => ... }`
dispatch is exhaustive by construction (the `_` arm makes it total over all
possible `&str` values) and compiles to a sequence of string comparisons —
there's no reflection-based command lookup table being built at runtime the
way a dynamic-language CLI framework might use. Because `Task` derives
`Debug`/similar traits at compile time (see Module 5), printing the list
costs only the formatting work itself; nothing about the struct's shape is
looked up dynamically. This is the same theme running through the whole
level: what looks like convenient high-level code — iterators, enums,
derives — is resolved and specialized entirely before the program starts
running.

## Stretch goals

- Add a `Priority` enum (`Low`/`Medium`/`High`) as a field on `Task` and sort
  the list by it.
- Add due dates using the `chrono` crate.
- Add a unit test for `storage::load_tasks`/`save_tasks` using a temp file
  (you'll formalize this properly with `cargo test` in
  [Level 2](../level-2/06-testing.md)).

Completing this project means you're ready for **Level 2 · Intermediate**.

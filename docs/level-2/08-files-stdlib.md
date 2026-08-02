# 08 · Files & the Standard Library

Almost every real program eventually needs to read a config file, write a
log, or process something bigger than fits comfortably in one `String` in
memory. Rust's standard library covers this through `std::fs` (whole-file
operations), `std::io` (streaming reads/writes), and `std::path` (working
with file paths portably across operating systems). This module ties them
together with the error-handling patterns from Module 05, since nearly every
I/O operation returns a `Result`.

## Writing and reading a whole file

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    fs::write("greeting.txt", "Hello, file system!\n")?;

    let contents = fs::read_to_string("greeting.txt")?;
    print!("{contents}");   // Hello, file system!

    fs::remove_file("greeting.txt")?;   // clean up
    Ok(())
}
```

```text
Hello, file system!
```

`fs::write` and `fs::read_to_string` are the simplest tools in the box — for
small files, reading the whole thing into memory at once is fine and far
less code than manual streaming.

## `File`, `OpenOptions`, and buffered I/O

For anything beyond "read/write the whole file at once," you work with a
`File` handle directly. Wrapping it in a `BufReader`/`BufWriter` batches
reads and writes into fewer actual system calls — without it, every
`.read_line()` or `write!()` could be its own syscall, which adds up fast on
larger files.

```rust
use std::fs::File;
use std::io::{self, BufRead, BufWriter, Write};

fn main() -> io::Result<()> {
    // Writing, buffered
    {
        let file = File::create("numbers.txt")?;
        let mut writer = BufWriter::new(file);
        for n in 1..=5 {
            writeln!(writer, "line {n}")?;   // writeln! works like println! but on a Write
        }
        // BufWriter flushes automatically when dropped, but explicit
        // `.flush()` is safer if you need to guarantee it happened before
        // the next line of code runs.
        writer.flush()?;
    }

    // Reading, buffered, line by line
    let file = File::open("numbers.txt")?;
    let reader = io::BufReader::new(file);
    for line in reader.lines() {
        let line = line?;   // each line is its own Result<String, io::Error>
        println!("read: {line}");
    }

    std::fs::remove_file("numbers.txt")?;
    Ok(())
}
```

```text
read: line 1
read: line 2
read: line 3
read: line 4
read: line 5
```

The inner block around the `File::create` and `BufWriter` forces the writer
to drop (and flush) before the file is reopened for reading — without that
scope, the buffered writer might not have flushed its contents to disk yet
when the read starts.

## `OpenOptions` for append mode and finer control

```rust
use std::fs::OpenOptions;
use std::io::Write;

fn main() -> std::io::Result<()> {
    // First write, creating the file
    {
        let mut file = OpenOptions::new()
            .create(true)
            .write(true)
            .truncate(true)
            .open("log.txt")?;
        writeln!(file, "first entry")?;
    }

    // Append a second entry without overwriting the first
    {
        let mut file = OpenOptions::new()
            .append(true)
            .open("log.txt")?;
        writeln!(file, "second entry")?;
    }

    let contents = std::fs::read_to_string("log.txt")?;
    print!("{contents}");

    std::fs::remove_file("log.txt")?;
    Ok(())
}
```

```text
first entry
second entry
```

`.create(true).write(true).truncate(true)` is the "start fresh" combination;
`.append(true)` (which implies write access) is the "add to the end without
touching existing content" combination — mixing these up is a common bug
(accidentally truncating a log file you meant to append to).

## Handling missing files without crashing

```rust
use std::fs;
use std::io::ErrorKind;

fn main() {
    match fs::read_to_string("does_not_exist.txt") {
        Ok(contents) => println!("{contents}"),
        Err(e) if e.kind() == ErrorKind::NotFound => {
            println!("file not found -- using defaults instead");
        }
        Err(e) => println!("unexpected error: {e}"),
    }
}
```

```text
file not found -- using defaults instead
```

`io::Error::kind()` returns an `ErrorKind` enum (`NotFound`,
`PermissionDenied`, `AlreadyExists`, and more) — matching on it lets you
react differently to "the file doesn't exist yet, that's fine" versus "I
don't have permission to read this," which is usually a real problem worth
surfacing.

## Working with `Path` and `PathBuf`

```rust
use std::path::{Path, PathBuf};

fn main() {
    let path = Path::new("/tmp/data/report.txt");

    println!("{:?}", path.file_name());      // Some("report.txt")
    println!("{:?}", path.extension());      // Some("txt")
    println!("{:?}", path.parent());         // Some("/tmp/data")
    println!("{}", path.exists());           // false (this path doesn't exist)

    // PathBuf is the owned, growable version of Path (like String vs &str)
    let mut owned: PathBuf = PathBuf::from("/tmp/data");
    owned.push("subfolder");
    owned.push("file.csv");
    println!("{}", owned.display());   // /tmp/data/subfolder/file.csv
}
```

```text
Some("report.txt")
Some("txt")
Some("/tmp/data")
false
/tmp/data/subfolder/file.csv
```

`Path`/`PathBuf` build paths with the correct separator for the current OS
(`/` on Unix, `\` on Windows) automatically — string-concatenating paths by
hand (`format!("{dir}/{file}")`) works on Unix but silently breaks on
Windows, so `.push()`/`.join()` is the portable habit to build.

## Reading standard input

```rust
use std::io::{self, Write};

fn main() {
    print!("Enter your name: ");
    io::stdout().flush().unwrap();   // print! doesn't auto-flush like println!

    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("failed to read line");

    let name = input.trim();   // strip the trailing newline read_line keeps
    println!("Hello, {name}!");
}
```

`.trim()` matters here — `read_line` includes the newline character in the
string it fills, so comparing or printing without trimming leaves a stray
`\n` that's easy to miss until it causes a confusing bug (like a name that
"looks right" but fails an equality check).

## Cheat sheet

| Task | Tool |
|------|------|
| Read a whole file into a `String` | `fs::read_to_string(path)` |
| Write a whole file at once | `fs::write(path, contents)` |
| Read line-by-line, efficiently | `BufReader::new(file).lines()` |
| Write many lines, efficiently | `BufWriter::new(file)` + `writeln!()` |
| Create-or-truncate, append instead | `OpenOptions::new().append(true)...` |
| Check *why* an I/O op failed | `err.kind() == ErrorKind::NotFound` etc. |
| Build a path portably | `PathBuf::from(...).push(...)` / `.join(...)` |
| Read one line from the user | `io::stdin().read_line(&mut buf)` |

## Exercise

Write a program that: creates a file `scores.txt`, writes three lines to it
in the form `name,score` using a `BufWriter`, then reopens the file with a
`BufReader` and reads it back line by line, splitting each line on `,` and
parsing the score into an `i32`. Print each name with its score, then print
the average score across all three lines. Handle a malformed line (missing
comma, or a non-numeric score) by skipping it and printing a warning instead
of crashing.

---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Rust Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```rust\nfn first_word(s: &str) -> &str { s.split(' ').next().unwrap_or(\"\") }\nfn main() {\n    let owned = String::from(\"hello world\");\n    let w = first_word(&owned);\n    println!(\"{w} {}\", owned.len());\n    let v: Vec<i32> = Vec::new();\n    println!(\"{:?} {}\", v.first(), v.is_empty());\n}\n```\n",
    "options": [
      "hello 5\nNone true",
      "hello 11\nSome(0) true",
      "hello world 11\nNone true",
      "hello 11\nNone true"
    ],
    "correctIndex": 3,
    "explanation": "Borrowed slices share the owner's data; first() on an empty Vec is None. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```rust\nuse std::collections::HashMap;\nfn main() {\n    let words = [\"a\", \"b\", \"a\", \"c\", \"a\", \"b\"];\n    let mut m: HashMap<&str, usize> = HashMap::new();\n    for w in words { *m.entry(w).or_default() += 1; }\n    let mut pairs: Vec<_> = m.into_iter().collect();\n    pairs.sort_by(|x, y| y.1.cmp(&x.1).then(x.0.cmp(y.0)));\n    println!(\"{:?}\", pairs);\n}\n```\n",
    "options": [
      "[(\"a\", 3), (\"c\", 1), (\"b\", 2)]",
      "[(\"c\", 1), (\"b\", 2), (\"a\", 3)]",
      "{\"a\": 3, \"b\": 2, \"c\": 1}",
      "[(\"a\", 3), (\"b\", 2), (\"c\", 1)]"
    ],
    "correctIndex": 3,
    "explanation": "HashMap iteration order is unspecified, so sort before printing. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```rust\nfn apply<F: Fn(i32) -> i32>(f: F, n: i32) -> i32 { f(f(n)) }\nfn make_adder(k: i32) -> impl Fn(i32) -> i32 { move |x| x + k }\nfn boxed(k: i32) -> Box<dyn Fn(i32) -> i32> { Box::new(move |x| x * k) }\nfn main() {\n    println!(\"{} {} {}\", apply(make_adder(3), 1), boxed(4)(5), apply(|x| x * x, 3));\n}\n```\n",
    "options": [
      "7 20 9",
      "7 20 81",
      "7 9 81",
      "5 20 81"
    ],
    "correctIndex": 1,
    "explanation": "impl Fn returns one concrete closure type; Box<dyn Fn> allows different closures at runtime. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```rust\n#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]\nenum Level { Low = 1, Mid, High }\nfn main() {\n    let mut v = vec![Level::High, Level::Low, Level::Mid];\n    v.sort();\n    println!(\"{:?} {} {}\", v, Level::Mid as i32, Level::Low < Level::High);\n}\n```\n",
    "options": [
      "[High, Mid, Low] 2 true",
      "[Low, Mid, High] 2 false",
      "[Low, Mid, High] 2 true",
      "[Low, Mid, High] 1 true"
    ],
    "correctIndex": 2,
    "explanation": "Derived Ord follows declaration order; discriminants continue counting. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```rust\nfn main() {\n    let data = vec![String::from(\"a\"), String::from(\"bb\")];\n    let lens: Vec<usize> = data.iter().map(|s| s.len()).collect();\n    let joined = data.join(\"-\");\n    let moved = data;\n    println!(\"{:?} {} {}\", lens, joined, moved.len());\n    let r: Result<u8, _> = \"300\".parse::<u8>();\n    println!(\"{}\", r.is_err());\n}\n```\n",
    "options": [
      "[1, 2] a-bb 0\ntrue",
      "[1, 2] a-bb 2\nfalse",
      "[2, 1] a-bb 2\ntrue",
      "[1, 2] a-bb 2\ntrue"
    ],
    "correctIndex": 3,
    "explanation": "Borrowing for iter() ends before the move; 300 does not fit in u8. Verified by compiling and running it."
  },
  {
    "question": "What does the borrow checker prevent in this code: let r; { let x = 1; r = &x; } use(r)?",
    "options": [
      "Type errors",
      "Overflow",
      "A dangling reference to x after its scope ends",
      "Slow code"
    ],
    "correctIndex": 2,
    "explanation": "References cannot outlive their referent."
  },
  {
    "question": "A function takes &mut Vec<T> while you hold an immutable reference to the vec. Why does it fail to compile?",
    "options": [
      "Lifetime of T",
      "A mutable borrow cannot coexist with a shared borrow",
      "Syntax",
      "Move"
    ],
    "correctIndex": 1,
    "explanation": "Aliasing XOR mutability."
  },
  {
    "question": "When should a library return Result instead of panicking?",
    "options": [
      "For any failure a caller might reasonably handle",
      "Always panic",
      "Only IO",
      "Never"
    ],
    "correctIndex": 0,
    "explanation": "Panic for bugs and broken invariants."
  },
  {
    "question": "What does the ? operator do on an Err?",
    "options": [
      "Ignores",
      "Returns early from the function, converting the error with From",
      "Panics",
      "Retries"
    ],
    "correctIndex": 1,
    "explanation": "It works on Option too."
  },
  {
    "question": "What is the difference between Rc<T> and Arc<T>?",
    "options": [
      "Arc is stack-only",
      "No difference",
      "Arc uses atomic counts and can be shared across threads; Rc cannot",
      "Rc is thread-safe"
    ],
    "correctIndex": 2,
    "explanation": "Rc is cheaper in single-threaded code."
  },
  {
    "question": "Why wrap shared mutable data in Arc<Mutex<T>>?",
    "options": [
      "Speed",
      "Arc shares ownership across threads and Mutex ensures exclusive mutable access",
      "Lifetimes",
      "Syntax"
    ],
    "correctIndex": 1,
    "explanation": "Lock scope should be short."
  },
  {
    "question": "What does an async fn do when called without .await?",
    "options": [
      "Blocks",
      "Creates a Future that does nothing until polled",
      "Panics",
      "Runs immediately"
    ],
    "correctIndex": 1,
    "explanation": "Futures are lazy."
  },
  {
    "question": "What is the safest way to run blocking work inside Tokio?",
    "options": [
      "Sleep",
      "tokio::task::spawn_blocking",
      "std::thread only",
      "Await it"
    ],
    "correctIndex": 1,
    "explanation": "Keep async threads free."
  },
  {
    "question": "What does #[derive(Deserialize)] with serde give you?",
    "options": [
      "Tests",
      "A server",
      "Generated code to parse formats such as JSON into your struct",
      "A database"
    ],
    "correctIndex": 2,
    "explanation": "Errors for missing or wrong-typed fields."
  },
  {
    "question": "What does cargo clippy -- -D warnings do in CI?",
    "options": [
      "Fails the build on lint warnings",
      "Formats",
      "Tests",
      "Docs"
    ],
    "correctIndex": 0,
    "explanation": "Keeps quality steady."
  },
  {
    "question": "What is the purpose of a SAFETY comment on an unsafe block?",
    "options": [
      "Compile",
      "To document why the invariants hold",
      "Law",
      "Style"
    ],
    "correctIndex": 1,
    "explanation": "Reviewers rely on it."
  },
  {
    "question": "Why prefer a multi-stage Docker build?",
    "options": [
      "Law",
      "Speed",
      "Style",
      "Ship only the compiled binary in a small image"
    ],
    "correctIndex": 3,
    "explanation": "Smaller attack surface."
  },
  {
    "question": "What does a Cargo workspace share between crates?",
    "options": [
      "Tests",
      "Source",
      "Docs",
      "The Cargo.lock and target directory"
    ],
    "correctIndex": 3,
    "explanation": "Consistent versions and faster builds."
  },
  {
    "question": "Which evidence best shows a Rust service is ready for production?",
    "options": [
      "Few lines",
      "Works locally",
      "It compiles",
      "Tests, clippy and fmt in CI, security audit, logs and metrics and a runbook"
    ],
    "correctIndex": 3,
    "explanation": "Operational readiness."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A ledger with typed errors

Implement apply for a small ledger: parse each line, update balances in a BTreeMap and report failures with a custom error enum, a Display impl and a From conversion so the ? operator works. Valid lines must still be applied when others fail.

**Starter code**

```rust
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

**Expected output**

```text
error: overdraft on bo: 50
error: bad amount: invalid digit found in string
error: bad line: ana transfer 5
error: bad line: ana deposit
ana: 70
bo: 0
errors: 4
```

<details><summary>Show solution and explanation</summary>

<pre><code>use std::collections::BTreeMap;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug, PartialEq)]
enum LedgerError {
    BadLine(String),
    BadAmount(ParseIntError),
    Overdraft { account: String, balance: i64 },
}

impl fmt::Display for LedgerError {
    fn fmt(&amp;self, f: &amp;mut fmt::Formatter) -&gt; fmt::Result {
        match self {
            LedgerError::BadLine(l) =&gt; write!(f, &quot;bad line: {l}&quot;),
            LedgerError::BadAmount(e) =&gt; write!(f, &quot;bad amount: {e}&quot;),
            LedgerError::Overdraft { account, balance } =&gt; write!(f, &quot;overdraft on {account}: {balance}&quot;),
        }
    }
}

impl From&lt;ParseIntError&gt; for LedgerError {
    fn from(e: ParseIntError) -&gt; Self { LedgerError::BadAmount(e) }
}

fn apply(balances: &amp;mut BTreeMap&lt;String, i64&gt;, line: &amp;str) -&gt; Result&lt;(), LedgerError&gt; {
    let parts: Vec&lt;&amp;str&gt; = line.split_whitespace().collect();
    if parts.len() != 3 { return Err(LedgerError::BadLine(line.to_string())); }
    let amount: i64 = parts[2].parse()?;
    let delta = match parts[1] {
        &quot;deposit&quot; =&gt; amount,
        &quot;withdraw&quot; =&gt; -amount,
        _ =&gt; return Err(LedgerError::BadLine(line.to_string())),
    };
    let bal = balances.entry(parts[0].to_string()).or_insert(0);
    if *bal + delta &lt; 0 {
        return Err(LedgerError::Overdraft { account: parts[0].to_string(), balance: *bal });
    }
    *bal += delta;
    Ok(())
}

fn main() {
    let lines = [&quot;ana deposit 100&quot;, &quot;bo deposit 50&quot;, &quot;ana withdraw 30&quot;, &quot;bo withdraw 80&quot;, &quot;cy deposit x1&quot;, &quot;ana transfer 5&quot;, &quot;ana deposit&quot;, &quot;bo withdraw 50&quot;];
    let mut balances = BTreeMap::new();
    let mut errors = 0;
    for l in lines {
        if let Err(e) = apply(&amp;mut balances, l) {
            errors += 1;
            println!(&quot;error: {e}&quot;);
        }
    }
    for (name, b) in &amp;balances { println!(&quot;{name}: {b}&quot;); }
    println!(&quot;errors: {errors}&quot;);
}
</code></pre>

<p>The match on the verb gives a signed delta, entry().or_insert(0) creates accounts on demand and the overdraft check happens before the balance changes. The From&lt;ParseIntError&gt; impl is what lets ? convert the parse error. Verified by compiling and running with Rust 1.97.</p>
</details>

### Count primes with worker threads

Share a counter in an Arc<Mutex<u32>> so worker threads claim chunks of 1000 numbers, count primes in each chunk and send results over an mpsc channel. The totals must match a single-worker run.

**Starter code**

```rust
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

**Expected output**

```text
primes below 10000: 1229
chunks: 10
first three: [(0, 168), (1000, 135), (2000, 127)]
same with one worker: true
```

<details><summary>Show solution and explanation</summary>

<pre><code>use std::sync::mpsc;
use std::sync::{Arc, Mutex};
use std::thread;

fn is_prime(n: u32) -&gt; bool {
    if n &lt; 2 { return false; }
    let mut d = 2;
    while d * d &lt;= n {
        if n % d == 0 { return false; }
        d += 1;
    }
    true
}

fn count_primes(limit: u32, workers: u32) -&gt; (u32, Vec&lt;(u32, u32)&gt;) {
    let next = Arc::new(Mutex::new(0u32));
    let chunk = 1000;
    let (tx, rx) = mpsc::channel();
    let mut handles = Vec::new();
    for id in 0..workers {
        let next = Arc::clone(&amp;next);
        let tx = tx.clone();
        handles.push(thread::spawn(move || {
            loop {
                let start = {
                    let mut g = next.lock().unwrap();
                    let s = *g;
                    *g += chunk;
                    s
                };
                if start &gt;= limit { break; }
                let end = (start + chunk).min(limit);
                let c = (start..end).filter(|&amp;n| is_prime(n)).count() as u32;
                tx.send((start, c)).unwrap();
            }
            id
        }));
    }
    drop(tx);
    let mut parts: Vec&lt;(u32, u32)&gt; = rx.iter().collect();
    for h in handles { h.join().unwrap(); }
    parts.sort();
    (parts.iter().map(|p| p.1).sum(), parts)
}

fn main() {
    let (total, parts) = count_primes(10_000, 4);
    println!(&quot;primes below 10000: {total}&quot;);
    println!(&quot;chunks: {}&quot;, parts.len());
    println!(&quot;first three: {:?}&quot;, &amp;parts[..3]);
    let (t1, _) = count_primes(10_000, 1);
    println!(&quot;same with one worker: {}&quot;, t1 == total);
}
</code></pre>

<p>The lock is held only while claiming the next chunk, the channel closes when every sender is dropped (the original tx is dropped explicitly) and sorting the results makes the output deterministic even though threads finish in any order. 1229 is the number of primes below 10,000. Verified by compiling and running with Rust 1.97.</p>
</details>

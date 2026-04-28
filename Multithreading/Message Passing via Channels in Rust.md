---
created: 2026-04-28T20:34:33+02:00
modified: 2026-04-28T20:34:47+02:00
---

# Message Passing via Channels in Rust

Channels in Rust follow the **"do not communicate by sharing memory; share memory by communicating"** philosophy. The standard library provides `std::sync::mpsc` (multi-producer, single-consumer), and the ecosystem offers more powerful alternatives.

---

## 1. Basic Channel (mpsc)

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send("hello from the thread!").unwrap();
    });

    let msg = rx.recv().unwrap();
    println!("Received: {msg}");
}
```

`tx` is the **sender** (transmitter), `rx` is the **receiver**. `send` is non-blocking; `recv` blocks until a message arrives.

---

## 2. Multiple Producers → One Consumer

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    for id in 0..4 {
        let tx = tx.clone(); // each thread gets its own sender clone
        thread::spawn(move || {
            tx.send(format!("worker-{id} reporting in")).unwrap();
        });
    }

    drop(tx); // drop the original so rx knows when ALL senders are gone

    for msg in rx {  // rx acts as an iterator — stops when channel closes
        println!("{msg}");
    }
}
```

> Cloning `tx` is the key — each clone is an independent sender. When **all** senders are dropped, `rx` iteration ends automatically.

---

## 3. Bounded / Synchronous Channel

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // Buffer capacity = 2: sender blocks when buffer is full
    let (tx, rx) = mpsc::sync_channel(2);

    let producer = thread::spawn(move || {
        for i in 0..5 {
            println!("Sending {i}...");
            tx.send(i).unwrap(); // blocks on 3rd item until consumer reads
            println!("Sent {i}");
        }
    });

    thread::sleep(Duration::from_millis(100)); // let producer fill the buffer

    for val in rx {
        println!("Consumed: {val}");
        thread::sleep(Duration::from_millis(50)); // slow consumer
    }

    producer.join().unwrap();
}
```

`sync_channel(n)` gives **back-pressure**: the producer can't race infinitely ahead of the consumer.

---

## Advanced Scenario 1: Pipeline (Chain of Channels)

Work flows through a series of transformation stages — a classic producer → stage → stage → consumer pattern:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // Stage 1: generate raw numbers
    let (s1_tx, s1_rx) = mpsc::channel::<i32>();
    // Stage 2: double them
    let (s2_tx, s2_rx) = mpsc::channel::<i32>();
    // Stage 3: filter evens only
    let (s3_tx, s3_rx) = mpsc::channel::<i32>();

    // Producer
    thread::spawn(move || {
        for i in 1..=10 { s1_tx.send(i).unwrap(); }
    });

    // Stage 2: double
    thread::spawn(move || {
        for val in s1_rx { s2_tx.send(val * 2).unwrap(); }
    });

    // Stage 3: keep only multiples of 4
    thread::spawn(move || {
        for val in s2_rx {
            if val % 4 == 0 { s3_tx.send(val).unwrap(); }
        }
    });

    // Consumer
    let results: Vec<i32> = s3_rx.iter().collect();
    println!("{results:?}"); // [4, 8, 12, 16, 20]
}
```

Each stage is a dedicated thread. Channel closure propagates automatically down the pipeline — no manual signaling needed.

---

## Advanced Scenario 2: Request / Response (Bidirectional)

Instead of a fire-and-forget channel, each message carries a **reply channel** so the sender can await a response:

```rust
use std::sync::mpsc;
use std::thread;

struct Request {
    input: u64,
    reply_tx: mpsc::Sender<u64>, // one-shot reply channel
}

fn main() {
    let (work_tx, work_rx) = mpsc::channel::<Request>();

    // Worker: receives requests, sends back results
    thread::spawn(move || {
        for req in work_rx {
            let result = fibonacci(req.input);
            req.reply_tx.send(result).unwrap();
        }
    });

    // Callers send a request and wait for their personal reply channel
    let handles: Vec<_> = [10u64, 20, 30].iter().map(|&n| {
        let work_tx = work_tx.clone();
        thread::spawn(move || {
            let (reply_tx, reply_rx) = mpsc::channel();
            work_tx.send(Request { input: n, reply_tx }).unwrap();
            let answer = reply_rx.recv().unwrap();
            println!("fib({n}) = {answer}");
        })
    }).collect();

    drop(work_tx);
    for h in handles { h.join().unwrap(); }
}

fn fibonacci(n: u64) -> u64 {
    match n { 0 => 0, 1 => 1, _ => fibonacci(n-1) + fibonacci(n-2) }
}
```

This is the foundation of **actor-style** concurrency in Rust — each "actor" owns its inbox channel and exposes a typed `Sender` handle.

---

## Quick Reference

| Pattern | API | Use when |
|---|---|---|
| Unbounded async | `mpsc::channel()` | Producer is naturally bounded |
| Bounded sync | `mpsc::sync_channel(n)` | Need back-pressure |
| Multi-producer | Clone `tx` | Fan-in from many threads |
| Pipeline | Chain channels | Staged data transformation |
| Request/reply | Embed reply `tx` in message | Actor-style RPC |

For **multi-consumer** (fan-out) or **select across channels**, look at the [`crossbeam-channel`](https://docs.rs/crossbeam-channel) crate, which adds `Receiver::clone()` and a `select!` macro.

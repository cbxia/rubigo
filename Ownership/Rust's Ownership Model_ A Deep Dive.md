---
created: 2026-05-01T23:02:16+02:00
modified: 2026-05-01T23:02:34+02:00
---

# Rust's Ownership Model: A Deep Dive

Rust's ownership system is its most distinctive feature — a compile-time memory management strategy that guarantees memory safety without a garbage collector. To understand it fully, we need to start from the ground up.

---

## 1. Stack vs. Heap: The Physical Foundation

Before ownership makes sense, you need to know *where* data lives.

### The Stack
- **Fixed-size**, **LIFO** (last-in, first-out) structure
- Allocation/deallocation is essentially free — just a pointer increment/decrement
- Data must have a size **known at compile time**
- Examples: integers, floats, booleans, tuples of fixed-size types, arrays

### The Heap
- **Dynamic**, arbitrarily sized memory
- Allocation requires asking the OS/allocator for a block → slower
- Data can grow or shrink at runtime
- Examples: `String`, `Vec<T>`, `Box<T>`, `HashMap<K, V>`

```rust
fn main() {
    // Stack: size known at compile time (i32 = 4 bytes, always)
    let x: i32 = 42;

    // Heap: String is a (ptr, len, capacity) triple on the stack,
    // but the actual character data lives on the heap
    let s: String = String::from("hello");

    // Stack: [i32; 3] is 12 bytes, fixed, lives entirely on the stack
    let arr: [i32; 3] = [1, 2, 3];

    // Heap: Vec's contents live on the heap; its length can change
    let v: Vec<i32> = vec![1, 2, 3];
}
```

```
Stack                    Heap
─────────────────────    ──────────────────────
x  │ 42            │
─────────────────────
s  │ ptr ──────────┼───► [ h | e | l | l | o ]
   │ len: 5        │
   │ cap: 5        │
─────────────────────
```

> **Why this matters for ownership:** Stack values are trivially copied (just memcpy a few bytes). Heap values carry an *owning pointer* — duplicating it naively would mean two things trying to free the same memory (a "double free" bug). Rust's ownership model prevents this entirely.

---

## 2. The Three Rules of Ownership

```
1. Each value in Rust has exactly one owner.
2. There can only be one owner at a time.
3. When the owner goes out of scope, the value is dropped.
```

These three rules, enforced at compile time, eliminate entire classes of bugs.

---

## 3. Move Semantics

When you assign a heap-allocated value to another variable, Rust **moves** it — the original binding becomes invalid.

### Example 1: Basic Move (Fails to Compile)

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // s1 is MOVED into s2

    println!("{}", s1);  // ❌ ERROR: value borrowed here after move
}
```

**Why it fails:**

```
Before move:           After `let s2 = s1`:
────────────           ────────────────────
s1 │ ptr → heap        s1 │ [INVALID - moved]
                        s2 │ ptr → heap  (same heap memory)
```

If both `s1` and `s2` were valid, Rust would try to `drop` (free) the heap memory twice when both go out of scope → **double-free**. Instead, Rust marks `s1` as moved. This is a compile error, not a runtime one.

---

### Example 2: Move into a Function

```rust
fn takes_ownership(s: String) {
    println!("{}", s);
}   // s is dropped here — heap memory freed

fn main() {
    let s = String::from("world");
    takes_ownership(s);
    println!("{}", s);  // ❌ ERROR: value used here after move
}
```

Passing `s` to the function **moves** it. The function now owns it. When the function returns, `s` is dropped. The caller's `s` is invalidated at the point of the call.

---

### Example 3: Copy Types — No Move, No Problem

Types that implement the `Copy` trait are **stack-only** and are bitwise-copied instead of moved. The original remains valid.

```rust
fn takes_copy(n: i32) {
    println!("{}", n);
}

fn main() {
    let x = 5;
    takes_copy(x);
    println!("{}", x);  // ✅ Fine! i32 is Copy
}
```

**Types that are `Copy`:** `i8`–`i128`, `u8`–`u128`, `f32`, `f64`, `bool`, `char`, `()`, tuples/arrays of `Copy` types.

**Types that are NOT `Copy`:** `String`, `Vec<T>`, `Box<T>`, anything with heap allocation (because shallow-copying the pointer would create double-free issues).

---

## 4. Borrowing and References

Moving is often too restrictive. Rust's solution: **borrow** a value temporarily via references.

### Immutable Borrow (`&T`)

```rust
fn print_length(s: &String) {  // borrows, does NOT take ownership
    println!("length = {}", s.len());
}   // s goes out of scope, but since it doesn't own the data, nothing is dropped

fn main() {
    let s = String::from("hello");
    print_length(&s);   // pass a reference
    println!("{}", s);  // ✅ s still valid — we only lent it
}
```

### Mutable Borrow (`&mut T`)

```rust
fn append_world(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");
    append_world(&mut s);
    println!("{}", s);  // "hello, world"
}
```

### Borrowing Rules (Enforced at Compile Time)

```
At any given time, you can have EITHER:
  • Any number of immutable references (&T), OR
  • Exactly ONE mutable reference (&mut T)
...but NEVER both simultaneously.
```

---

### Example 4: Borrow Conflict (Fails to Compile)

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;      // immutable borrow
    let r2 = &s;      // another immutable borrow — fine so far
    let r3 = &mut s;  // ❌ ERROR: cannot borrow `s` as mutable because
                      //          it is also borrowed as immutable

    println!("{}, {}, {}", r1, r2, r3);
}
```

**Why:** If `r3` could mutate `s` while `r1`/`r2` exist, they'd observe data changing under them — a **data race**. Rust forbids this at compile time.

---

### Example 5: Non-Lexical Lifetimes (NLL) — The Nuance

Rust is smart about when borrows *actually end*. Since Rust 2018, borrows end at their **last use**, not at the end of the enclosing block:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // r1 and r2 are no longer used after this line ← NLL ends borrow here

    let r3 = &mut s;  // ✅ Fine! r1 and r2 are already "done"
    r3.push_str("!");
    println!("{}", r3);
}
```

---

## 5. Drop Behavior and RAII

Rust uses **RAII** (Resource Acquisition Is Initialization): resources are tied to objects and freed when those objects are dropped.

```rust
struct MyBuffer {
    data: Vec<u8>,
}

impl Drop for MyBuffer {
    fn drop(&mut self) {
        println!("MyBuffer dropped! Freeing {} bytes.", self.data.len());
    }
}

fn main() {
    let buf = MyBuffer { data: vec![0u8; 1024] };
    println!("Buffer created.");
    // buf goes out of scope here → Drop::drop() is called automatically
}
```

**Output:**
```
Buffer created.
MyBuffer dropped! Freeing 1024 bytes.
```

Drop order follows **reverse declaration order** within a scope:

```rust
fn main() {
    let a = MyBuffer { data: vec![0; 1] };  // created first
    let b = MyBuffer { data: vec![0; 2] };  // created second
    // b is dropped first (LIFO), then a
}
```

---

### Example 6: Drop and Partial Moves (Edge Case)

```rust
struct Pair {
    first: String,
    second: String,
}

fn main() {
    let p = Pair {
        first: String::from("foo"),
        second: String::from("bar"),
    };

    let first = p.first;   // partial move: `first` field is moved out
    println!("{}", first); // ✅ fine

    println!("{}", p.second); // ✅ fine — second wasn't moved
    println!("{}", p.first);  // ❌ ERROR: value partially moved
    // Also: you can't use `p` as a whole anymore either
}
```

Once a field is partially moved out, the struct can no longer be used as a whole, and neither can the moved field through the original binding.

---

## 6. Lifetimes: Borrow Checking Across Scope Boundaries

Lifetimes are Rust's way of ensuring references don't outlive the data they point to.

### Example 7: Dangling Reference (Fails to Compile)

```rust
fn dangle() -> &String {  // ❌ ERROR: missing lifetime specifier
    let s = String::from("hello");
    &s  // we return a reference to s...
}   // ...but s is dropped here! The reference would dangle.

fn main() {
    let r = dangle();
}
```

In C/C++, this compiles and produces **undefined behavior** at runtime. Rust catches it at compile time.

**The fix:** Return the `String` itself, transferring ownership to the caller.

```rust
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // ownership moved to caller — no drop
}
```

---

### Example 8: Explicit Lifetimes

When a function takes multiple references and returns one, Rust needs to know which input the output is tied to:

```rust
// 'a means: "the output reference lives at most as long as both inputs"
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long string");
    let result;
    {
        let s2 = String::from("xy");
        result = longest(s1.as_str(), s2.as_str());
        println!("Longest: {}", result);  // ✅ fine — result used inside s2's scope
    }
    // println!("{}", result);  // ❌ would fail — s2 dropped, result might point to it
}
```

---

## 7. Smart Pointers and Ownership Patterns

### `Box<T>` — Heap Allocation with Single Ownership

```rust
fn main() {
    let b = Box::new(5);   // 5 lives on the heap
    println!("{}", b);     // auto-dereferenced
    // b dropped here → heap memory freed
}
```

### `Rc<T>` — Reference Counted Shared Ownership (Single-Threaded)

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("shared"));
    let b = Rc::clone(&a);  // increments reference count — no heap copy
    let c = Rc::clone(&a);

    println!("count = {}", Rc::strong_count(&a));  // 3
    drop(b);
    println!("count = {}", Rc::strong_count(&a));  // 2
    // Heap memory freed only when count reaches 0
}
```

### `RefCell<T>` — Interior Mutability (Borrow Rules Checked at Runtime)

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);

    let r1 = data.borrow();      // immutable borrow at runtime
    // let r2 = data.borrow_mut(); // ❌ would PANIC at runtime (not compile time!)
    println!("{:?}", r1);
}
```

`RefCell` moves borrow checking to runtime — useful when the compiler can't statically verify safety but you know it's correct. Violations become panics, not compile errors.

---

## 8. Putting It All Together: A Realistic Example

```rust
use std::collections::HashMap;

/// Returns the longest word in each category.
/// Borrows the map — does not take ownership.
fn longest_per_category<'a>(
    categories: &'a HashMap<String, Vec<String>>,
) -> HashMap<&'a str, &'a str> {
    categories
        .iter()
        .filter_map(|(category, words)| {
            words.iter()
                 .max_by_key(|w| w.len())  // borrow each word
                 .map(|longest| (category.as_str(), longest.as_str()))
        })
        .collect()
}

fn main() {
    let mut db: HashMap<String, Vec<String>> = HashMap::new();
    db.insert("fruits".into(), vec!["apple".into(), "banana".into(), "fig".into()]);
    db.insert("colors".into(), vec!["red".into(), "ultraviolet".into(), "blue".into()]);

    let result = longest_per_category(&db);  // db is borrowed, not moved
    for (cat, word) in &result {
        println!("{}: {}", cat, word);
    }
    // db still usable here — we only borrowed it
    println!("Total categories: {}", db.len());
}
```

**Key observations:**
- `db` is borrowed by `longest_per_category` — it's still usable after the call
- The returned `HashMap` holds references (`&str`) into `db`, so the lifetime annotation `'a` ensures the result can't outlive `db`
- All of this is verified at compile time — zero runtime overhead for the safety guarantees

---

## Summary Table

| Concept | Stack | Heap (`String`, `Vec`, etc.) |
|---|---|---|
| Assignment | Copies (if `Copy`) | Moves (invalidates original) |
| Function pass | Copies (if `Copy`) | Moves (unless you pass `&T`) |
| Drop | Implicit, free | Calls `drop()`, frees heap memory |
| Multiple owners | N/A (copied freely) | Use `Rc<T>` / `Arc<T>` |
| Shared mutation | N/A | Use `RefCell<T>` / `Mutex<T>` |

Rust's ownership model trades a steeper learning curve for a profound guarantee: **if it compiles, it is memory safe** — no dangling pointers, no double frees, no data races, no use-after-free. The compiler is your memory safety auditor, working for free at compile time.

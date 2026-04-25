---
created: 2026-04-25T12:23:22+02:00
modified: 2026-04-25T16:58:19+02:00
---

## Rust Closures: Internal Workings

### What is a Closure?

A closure in Rust is an **anonymous function that can capture variables from its surrounding scope**. Internally, the compiler transforms every closure into an anonymous struct that implements one or more of the `Fn`, `FnMut`, or `FnOnce` traits.

---

### How the Compiler Sees a Closure

When you write a closure, Rust synthesizes a hidden struct to hold the captured variables. For example:

```rust
let x = 5;
let add_x = |n| n + x;
```

The compiler roughly transforms this into:

```rust
struct __Closure<'a> {
    x: &'a i32,   // captured by reference (default)
}

impl<'a> Fn(i32) -> i32 for __Closure<'a> {
    fn call(&self, n: i32) -> i32 {
        n + *self.x
    }
}

// Lifetime 参数的意义：隐含struct（即closure）的生命周期不得超过其field x（即被捕获变量）的生命周期。这在多线程函数里如果二者处于同一作用域内是可以满足的，因为二者具有相同的生命周期。在多线程函数里，往往不可满足这个条件，除非被捕获变量具有全局生命周期。
```

The closure value `add_x` **is** this anonymous struct instance.

---

### Capture Modes (Without `move`)

By default, Rust's borrow checker chooses the **least restrictive capture mode** needed:

| Usage in closure body | Capture mode | Trait implemented |
|---|---|---|
| Only reads the variable | `&T` (shared ref) | `Fn` |
| Mutates the variable | `&mut T` (mutable ref) | `FnMut` |
| Consumes / moves the variable | `T` (owned) | `FnOnce` |

```rust
let s = String::from("hello");
let name = 42;

let c1 = || println!("{name}");       // captures name: &i32  → Fn
let c2 = || println!("{s}");          // captures s: &String  → Fn
let c3 = || drop(s);                  // captures s: String   → FnOnce
```

The compiler infers these automatically — no annotation needed.

---

### The `Fn` Trait Hierarchy

```
FnOnce   ← base trait (can be called at least once, may consume captures)
  └── FnMut  ← called multiple times with &mut self
        └── Fn    ← called multiple times with &self
```

- Every closure implements at least `FnOnce`.
- If it doesn't consume captures, it also implements `FnMut`.
- If it doesn't mutate captures, it also implements `Fn`.

---

### Without `move` — Borrowing from the Environment

```rust
fn main() {
    let data = vec![1, 2, 3];

    let print_data = || println!("{:?}", data); // borrows &data

    print_data(); // OK
    print_data(); // OK — borrow is shared, reusable

    println!("{:?}", data); // still valid: closure only borrowed it
}
```

**Internally:**
```rust
struct __Closure<'a> {
    data: &'a Vec<i32>,   // just a reference
}
```

The closure struct holds a **reference** into the enclosing stack frame. This is cheap (pointer-sized) but the closure **cannot outlive** the scope where `data` lives.

---

### With `move` — Transferring Ownership

`move` forces the closure to **take ownership** of every captured variable, regardless of how it uses them:

```rust
fn main() {
    let data = vec![1, 2, 3];

    let print_data = move || println!("{:?}", data); // owns data

    // println!("{:?}", data); // ← compile error: data was moved

    print_data();
    print_data(); // still OK: data is inside the closure, not consumed
}
```

**Internally:**
```rust
struct __Closure {
    data: Vec<i32>,   // owned copy, no lifetime parameter needed
}
```

The lifetime annotation disappears — the closure is now **self-contained**.

---

### Why Use `move`?

**1. Sending closures across threads** — threads may outlive the current scope:
```rust
let msg = String::from("hello");

std::thread::spawn(move || {        // msg must be owned
    println!("{msg}");
});
// msg is gone from here
```

**2. Returning closures from functions** — the stack frame will be gone:
```rust
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |n| n + x    // x is owned by the closure, not borrowed from stack
}
```

Without `move` here, `x` would be a dangling reference after the function returns.

---

### Side-by-side Summary

```rust
let x = String::from("hi");

// Without move: borrows x
let c1 = || println!("{x}");     // struct holds &String
                                  // x still usable here
                                  // closure tied to x's lifetime

// With move: owns x
let c2 = move || println!("{x}");  // struct holds String
                                    // x is gone from this scope
                                    // closure is 'static if all captures are 'static
```

| | Without `move` | With `move` |
|---|---|---|
| Capture storage | Reference (`&T` / `&mut T`) | Owned value (`T`) |
| Struct has lifetime? | Yes (`'a`) | No |
| Original var usable? | Yes (if shared borrow) | No (moved away) |
| Can cross thread/scope boundary? | No | Yes (if `T: Send`) |
| Typical trait | `Fn` / `FnMut` | `Fn` / `FnMut` / `FnOnce` |

---

## Key Takeaway

Rust closures are **compiler-generated structs**. The `move` keyword controls whether those structs store **references into the enclosing scope** or **own their captured data outright**. Everything else — which `Fn` trait is implemented, whether the closure is `Send`, whether it has a lifetime — flows naturally from that single decision.

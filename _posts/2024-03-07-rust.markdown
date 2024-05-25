---
layout: post
title: "Rust Log 🦀"
date: 2024-05-23 00:38:17 -0800
categories: jekyll update
---

i caved

# May 23rd:

- Rust is a very opinionated programming language! As a newbie, I feel like I need to understand the Rust philosophy of approaching memory without garbage collector/fighting borrow checker in order to actually get it. To me, Rust philosophy is like: “worry about this now, compile time will be slow” but “runtime will be super fast”.
- Box::new(5), pointer that allocates memory on the heap for an integer, 5.
- Detail in error messages matches the error complexity! Rust compiler is nice (e.g. if you use a variable that has been moved).

```rust
fn main() {
    let x = Box::new(5);

    let mut y = x; // mutable reference to x.

    *y = 4;

    assert_eq!(*y, 4);

    println!("Success!");
}

  Compiling playground v0.0.1 (/playground)
error[E0382]: borrow of moved value: `x`
 --> src/main.rs:9:5
  |
3 |     let x = Box::new(5);
  |         - move occurs because `x` has type `Box<i32>`, which does not implement the `Copy` trait
4 |
5 |     let mut y = x; // mutable reference to x.
  |                 - value moved here
...
9 |     assert_eq!(*x, 4);
  |     ^^^^^^^^^^^^^^^^^ value borrowed here after move
  |
  = note: this error originates in the macro `assert_eq` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider cloning the value if the performance cost is acceptable
  |
5 |     let mut y = x.clone(); // mutable reference to x.
  |                  ++++++++

For more information about this error, try `rustc --explain E0382`.
error: could not compile `playground` (bin "playground") due to 1 previous error


- string literals (&s aka string slice) vs. String types (String::from("hello world") using a string literal, stored as vector of bytes) are a thing!
- .tostring() converts a string literal to a String type
```

- &mut vs ref: &mut is used to create a mutable reference while ref is used to create a reference in destructuring (for example, a tuple)

```rust
fn main() {
    let t = (String::from("hello"), String::from("world"));

    // Using ref to borrow the elements of the tuple
    let (ref s1, ref s2) = t;

    // Now s1 and s2 are references to the elements of t
    println!("{}, {}", s1, s2); // Output: "hello", "world"
```

- You can only borrow once (cannot create multiple mutable references for the same variable. You can create mutliple immutable references, however. This is denoted by &s. This is an example of creating multiple mutable references, and is wrong:

```rust

fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;

    println!("{}, {}", r1, r2);

    println!("Success!");
}
```

Why does Rust only allow one copy of mutable references at a time? 
Data race will occur if not. A data race is when two or more threads access the same memory location concurrently. Multiple versions for the same reference. Rust core philosophy: multiple threads can read the data (immutable references) or only one thread can modify the data at any time (single mutable reference). 

You are allowed to borrow a mutable object as an immutable one. 
```rust 
fn main() {
    let mut s = String::from("hello, ");

    borrow_object(&s);
    
    s.push_str("world");

    println!("Success!");
}

fn borrow_object(s: &String) {}

```


### May 25th 
Modules and crates today. 

- Modules allow for limited scope, avoid naming conflicts. Group related code. 
- src/main.rs  = crate root for binary crate 
- src/lib.rs = crate root for library crate 
- you need to preface with use crate::sibling_module if you have sibling modules (modules at the same level) and need to access the sibling_module 
- If you have a root crate with submodules, at the root file, you need to declare your submodules at the top 
    Example: 
    front_of_house 
     -- hosting.rs 
     -- mod.rs 
     -- serving.rs
    
    mod.rs would have:
    ``` rust
    pub mod hosting; 
  pub mod serving;   
    ```

    ```rust 
    mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```

  - add_to_waitlist is defined in the same root crate as eat_at_restaurant so you can use the "crate" word to specify the same root crate 


More examples: 
Correct: 
```rust 
  
pub mod foo {
  pub mod bar {
    pub fn b() { println!("b");  }
  }
  pub fn a() { bar::b(); }
}

fn main() {
  foo::a();
}
```

it is valid to use bar::b() just to refer to b() because "bar" and "b" are both public. You can refer to functions in the same module to refer to the items in submodules. 
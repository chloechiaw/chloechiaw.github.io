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


How to implement Eq and Hash for my own structs to use them as a HashMap key?

I have two structs, `A` and `B`, and I want to use a `HashMap<A, B>`. I have a piece of code like this:

```rust
use std::collections::HashMap;

pub struct A {
    x: i32,
    y: i32,
    title: String,
}

pub struct B {
    a: u32,
    b: u32,
}

fn main() {
    let map = HashMap::new();
    map.insert(
        A {
            x: 10,
            y: 20,
            title: "test".to_string(),
        },
        B { a: 1, b: 2 },
    );
}
```

But the compiler gives me these errors:

```none
error[E0277]: the trait bound `A: std::cmp::Eq` is not satisfied
  --> src/main.rs:16:9
   |
16 |     map.insert(
   |         ^^^^^^ the trait `std::cmp::Eq` is not implemented for `A`

error[E0277]: the trait bound `A: std::hash::Hash` is not satisfied
  --> src/main.rs:16:9
   |
16 |     map.insert(
   |         ^^^^^^ the trait `std::hash::Hash` is not implemented for `A`
```

I know that I must implement these traits, but after hours of searching the web, I have found nothing about implementing them.

My actual code is more complicated, and my structs contain other structs (I've edited the code).

I've implemented the `Hash` trait:

```rust
impl std::hash::Hash for A {
    fn hash<H>(&self, state: &mut H)
    where
        H: std::hash::Hasher,
    {
        state.write_i32(self.x);
        state.finish();
    }
}
```

I made an implementation for `PartialEq` also:

```rust
impl PartialEq for A {
    fn eq(&self, other: &A) -> bool {
        self.x == other.x
    }
}
```

But the compiler continues to complain, this time about `Eq`:

```none
error[E0277]: the trait bound `A: std::cmp::Eq` is not satisfied
  --> src/main.rs:16:9
   |
16 |     map.insert(
   |         ^^^^^^ the trait `std::cmp::Eq` is not implemented for `A`
```

How can I implement `Eq`? Why is there no implementation in the docs?

answer1

`Eq` is what we call a marker trait: it has no method on its own, it is just a way for the programmer to express that the struct verifies a certain property. You can implement it like this:

```rust
impl Eq for Application {}
```

Or alternatively, use `#[derive(Eq)]` on top of the `Application` declaration

`Eq` is a trait bound by `PartialEq`. This means that you can implement it only on structs that also implement `PartialEq` (which is the case here). By implementing `Eq`, you make the promise that your implementation of `PartialEq` is reflexive (see the docs for what it means).

answer2

You can have the compiler derive these instances for you by inserting the following before your struct declaration:

```rust
#[derive(PartialEq, Eq, Hash)]
pub struct A {
    // ...
}
```

You could also implement them manually instead. If you want to do that, you should read the documentation on [traits](https://doc.rust-lang.org/stable/book/second-edition/ch10-02-traits.html), [`Eq`](https://doc.rust-lang.org/std/cmp/trait.Eq.html) and [`Hash`](https://doc.rust-lang.org/std/hash/trait.Hash.html).

answer3

This is how the Rust documentation says you write your own implementation of `Hash`:

```rust
use std::hash::{Hash, Hasher};

struct Person {
    id: u32,
    name: String,
    phone: u64,
}

impl Hash for Person {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
        self.phone.hash(state);
    }
}
```

Source: https://doc.rust-lang.org/std/hash/trait.Hash.html


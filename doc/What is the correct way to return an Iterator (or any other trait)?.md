What is the correct way to return an Iterator (or any other trait)?

The following Rust code compiles and runs without any issues.

```rust
fn main() {
    let text = "abc";
    println!("{}", text.split(' ').take(2).count());
}
```

After that, I tried something like this .... but it didn't compile

```rust
fn main() {
    let text = "word1 word2 word3";
    println!("{}", to_words(text).take(2).count());
}

fn to_words(text: &str) -> &Iterator<Item = &str> {
    &(text.split(' '))
}
```

The main problem is that I'm not sure what return type the function `to_words()` should have. The compiler says:

```none
error[E0599]: no method named `count` found for type `std::iter::Take<std::iter::Iterator<Item=&str>>` in the current scope
 --> src/main.rs:3:43
  |
3 |     println!("{}", to_words(text).take(2).count());
  |                                           ^^^^^
  |
  = note: the method `count` exists but the following trait bounds were not satisfied:
          `std::iter::Iterator<Item=&str> : std::marker::Sized`
          `std::iter::Take<std::iter::Iterator<Item=&str>> : std::iter::Iterator`
```

What would be the correct code to make this run? .... and where is my knowledge gap?

I've found it useful to let the compiler guide me:

```rust
fn to_words(text: &str) { // Note no return type
    text.split(' ')
}
```

Compiling gives:

```none
error[E0308]: mismatched types
 --> src/lib.rs:5:5
  |
5 |     text.split(' ')
  |     ^^^^^^^^^^^^^^^ expected (), found struct `std::str::Split`
  |
  = note: expected type `()`
             found type `std::str::Split<'_, char>`
help: try adding a semicolon
  |
5 |     text.split(' ');
  |                    ^
help: try adding a return type
  |
3 | fn to_words(text: &str) -> std::str::Split<'_, char> {
  |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Following the compiler's suggestion and copy-pasting that as my return type (with a little cleanup):

```rust
use std::str;

fn to_words(text: &str) -> str::Split<'_, char> {
    text.split(' ')
}
```

The problem is that you cannot return a trait like `Iterator` because a trait doesn't have a size. That means that Rust doesn't know how much space to allocate for the type. You [cannot return a reference to a local variable, either](https://stackoverflow.com/questions/32682876/is-there-any-way-to-return-a-reference-to-a-variable-created-in-a-function), so returning `&dyn Iterator` is a non-starter.

### Impl trait

As of Rust 1.26, you can use [`impl trait`](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md):

```rust
fn to_words<'a>(text: &'a str) -> impl Iterator<Item = &'a str> {
    text.split(' ')
}

fn main() {
    let text = "word1 word2 word3";
    println!("{}", to_words(text).take(2).count());
}
```

There are restrictions on how this can be used. You can only return a single type (no conditionals!) and it must be used on a free function or an inherent implementation.

### Boxed

If you don't mind losing a little bit of efficiency, you can return a `Box<dyn Iterator>`:

```rust
fn to_words<'a>(text: &'a str) -> Box<dyn Iterator<Item = &'a str> + 'a> {
    Box::new(text.split(' '))
}

fn main() {
    let text = "word1 word2 word3";
    println!("{}", to_words(text).take(2).count());
}
```

This is the primary option that allows for *dynamic dispatch*. That is, the exact implementation of the code is decided at run-time, rather than compile-time. That means this is suitable for cases where you need to return more than one concrete type of iterator based on a condition.

### Newtype

```rust
use std::str;

struct Wrapper<'a>(str::Split<'a, char>);

impl<'a> Iterator for Wrapper<'a> {
    type Item = &'a str;

    fn next(&mut self) -> Option<&'a str> {
        self.0.next()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        self.0.size_hint()
    }
}

fn to_words(text: &str) -> Wrapper<'_> {
    Wrapper(text.split(' '))
}

fn main() {
    let text = "word1 word2 word3";
    println!("{}", to_words(text).take(2).count());
}
```

### Type alias

As [pointed out by reem](https://stackoverflow.com/questions/27535289/correct-way-to-return-an-iterator#comment43509185_27535594)

```rust
use std::str;

type MyIter<'a> = str::Split<'a, char>;

fn to_words(text: &str) -> MyIter<'_> {
    text.split(' ')
}

fn main() {
    let text = "word1 word2 word3";
    println!("{}", to_words(text).take(2).count());
}
```

### Dealing with closures

When `impl Trait` isn't available for use, closures make things more complicated. Closures create anonymous types and these cannot be named in the return type:

```rust
fn odd_numbers() -> () {
    (0..100).filter(|&v| v % 2 != 0)
}
found type `std::iter::Filter<std::ops::Range<{integer}>, [closure@src/lib.rs:4:21: 4:36]>`
```

In certain cases, these closures can be replaced with functions, which can be named:

```rust
fn odd_numbers() -> () {
    fn f(&v: &i32) -> bool {
        v % 2 != 0
    }
    (0..100).filter(f as fn(v: &i32) -> bool)
}
found type `std::iter::Filter<std::ops::Range<i32>, for<'r> fn(&'r i32) -> bool>`
```

And following the above advice:

```rust
use std::{iter::Filter, ops::Range};

type Odds = Filter<Range<i32>, fn(&i32) -> bool>;

fn odd_numbers() -> Odds {
    fn f(&v: &i32) -> bool {
        v % 2 != 0
    }
    (0..100).filter(f as fn(v: &i32) -> bool)
}
```

### Dealing with conditionals

If you need to conditionally choose an iterator, refer to [Conditionally iterate over one of several possible iterators](https://stackoverflow.com/q/29760668/155423).


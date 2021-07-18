String vs &str iterators

I'm trying to write a function which accepts a list of tokens. But I'm having problems making it general enough to handle two pretty similar calls:

```rust
let s = String::from("-abc -d --echo");

parse( s.split_ascii_whitespace() );
parse( std::env::args() );
```

- [`String::split_ascii_whitespace()`](https://doc.rust-lang.org/std/str/struct.SplitAsciiWhitespace.html#impl-Iterator) returns `std::str:SplitAsciiWhitespace` which implements `Iterator<Item=&'a str>`.
- [`std::env::args()`](https://doc.rust-lang.org/std/env/struct.Args.html#impl-Iterator) returns `std::env::Args` which implements `Iterator<Item=String>`.

Is there a way for me to write a function signature for `parse` that will accept both methods?

My solution right now requires duplicating function bodies:

```rust
fn main() {
    let s = String::from("-abc -d --echo");

    parse_args( s.split_ascii_whitespace() );
    parse_env( std::env::args() );
}

fn parse_env<I: Iterator<Item=String>>(mut it: I) {
    loop {
        match it.next() {
            None => return,
            Some(s) => println!("{}",s),
        }
    }
}

fn parse_args<'a, I: Iterator<Item=&'a str>>(mut it: I) {
    loop {
        match it.next() {
            None => return,
            Some(s) => println!("{}",s),
        }
    }
}
```

If not possible, then some advice on how to use the traits so the functions can use the same name would be nice.

answer1

You can require the item type to be `AsRef<str>`, which will include both `&str` and `String`:

```rust
fn parse<I>(mut it: I)
where
    I: Iterator,
    I::Item: AsRef<str>,
{
    loop {
        match it.next() {
            None => return,
            Some(s) => println!("{}", s.as_ref()),
        }
    }
}
```

answer2

Depending on your use case, you could try:

```rust
fn main() {
    let s = String::from("-abc -d --echo");

    parse( s.split_ascii_whitespace() );
    parse( std::env::args() );
}

fn parse<T: std::borrow::Borrow<str>, I: Iterator<Item=T>>(mut it: I) {
    loop {
        match it.next() {
            None => return,
            Some(s) => println!("{}",s.borrow()),
        }
    }
}
```

I used `Borrow` as a means to get to a `&str`, but your concrete use case may be served by other, possibly custom, traits.

Notes:

`Borrow` and `AsRef` are confusingly similar. Historically, the reason for the introduction of `Borrow` was that hash maps required consistency between the `Hash`, `Eq` and `Ord` implementations of the borrowed value and the original value. Since `AsRef` did not have these additional requirements, a new trait was needed. In this use case the additional requirements are irrelevant, though – we just want to convert whatever item type we get to an `&str`, so I believe `AsRef` is more appropriate here, but I'll admit that the difference is rather philosophical in this particular case. 

`AsRef<T>` is a well-known idiom with many uses in the stdlib and elsewhere. For example `AsRef<Path>` is used to [accept paths](https://doc.rust-lang.org/std/fs/struct.File.html#method.open) in the many ways they can be created (from strings, `OsStr`, `PathBuf`, etc.), `AsRef<OsStr>` for things where an OS string is needed such as [command line](https://doc.rust-lang.org/std/process/struct.Command.html#method.arg), and so on. `AsRef<str>` will be familiar to anyone who encountered those, while `Borrow<str>` will falsely indicate additional requirements. 
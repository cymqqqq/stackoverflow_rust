Is there any way to return a reference to a variable created in a function?

I want to write a program that will write a file in 2 steps. It is likely that the file may not exist before the program is run. The filename is fixed.

The problem is that `OpenOptions.new().write()` can fail. In that case, I want to call a custom function `trycreate()`. The idea is to create the file instead of opening it and return a handle. Since the filename is fixed, `trycreate()` has no arguments and I cannot set a lifetime of the returned value.

How can I resolve this problem?

```rust
use std::io::Write;
use std::fs::OpenOptions;
use std::path::Path;

fn trycreate() -> &OpenOptions {
    let f = OpenOptions::new().write(true).open("foo.txt");
    let mut f = match f {
        Ok(file)  => file,
        Err(_)  => panic!("ERR"),
    };
    f
}

fn main() {
    {
        let f = OpenOptions::new().write(true).open(b"foo.txt");
        let mut f = match f {
            Ok(file)  => file,
            Err(_)  => trycreate("foo.txt"),
        };
        let buf = b"test1\n";
        let _ret = f.write(buf).unwrap();
    }
    println!("50%");
    {
        let f = OpenOptions::new().append(true).open("foo.txt");
        let mut f = match f {
            Ok(file)  => file,
            Err(_)  => panic!("append"),
        };
        let buf = b"test2\n";
        let _ret = f.write(buf).unwrap();
    }
    println!("Ok");
}
```

answer1

No, you cannot return a reference to a variable that is owned by a function. This applies if you created the variable or if you took ownership of the variable as a function argument.

## Solutions

Instead of trying to return a reference, return an owned object. `String` instead of `&str`, `Vec<T>` instead of `&[T]`, `T` instead of `&T`, etc.

If you took ownership of the variable via an argument, try taking a (mutable) reference instead and then returning a reference of the same lifetime.

In rare cases, you can use unsafe code to return the owned value *and* a reference to it. This has a number of delicate requirements you must uphold to ensure you don't cause undefined behavior or memory unsafety.

See also:

- [Proper way to return a new string in Rust](https://stackoverflow.com/q/43079077/155423)
- [Return local String as a slice (&str)](https://stackoverflow.com/q/29428227/155423)
- [Why can't I store a value and a reference to that value in the same struct?](https://stackoverflow.com/q/32300132/155423)

## Deeper answer

[fjh is absolutely correct](https://stackoverflow.com/a/32683137/155423), but I want to comment a bit more deeply and touch on some of the other errors with your code.

Let's start with a smaller example of returning a reference and look at the errors:

```rust
fn try_create<'a>() -> &'a String {
    &String::new()
}
```

**Rust 2015**

```none
error[E0597]: borrowed value does not live long enough
 --> src/lib.rs:2:6
  |
2 |     &String::new()
  |      ^^^^^^^^^^^^^ temporary value does not live long enough
3 | }
  | - temporary value only lives until here
  |
note: borrowed value must be valid for the lifetime 'a as defined on the function body at 1:15...
 --> src/lib.rs:1:15
  |
1 | fn try_create<'a>() -> &'a String {
  |               ^^
```

**Rust 2018**

```none
error[E0515]: cannot return reference to temporary value
 --> src/lib.rs:2:5
  |
2 |     &String::new()
  |     ^-------------
  |     ||
  |     |temporary value created here
  |     returns a reference to data owned by the current function
```

> Is there any way to return a reference from a function without arguments?

Technically "yes", but for what you want, "no".

A reference points to an existing piece of memory. In a function with no arguments, the only things that could be referenced are global constants (which have the lifetime `&'static`) and local variables. I'll ignore globals for now.

In a language like C or C++, you could actually take a reference to a local variable and return it. However, as soon as the function returns, there's **no guarantee** that the memory that you are referencing continues to be what you thought it was. It might stay what you expect for a while, but eventually the memory will get reused for something else. As soon as your code looks at the memory and tries to interpret a username as the amount of money left in the user's bank account, problems will arise!

This is what Rust's lifetimes prevent - you aren't allowed to use a reference beyond how long the referred-to value is valid at its current memory location.

See also:

- [Is it possible to return either a borrowed or owned type in Rust?](https://stackoverflow.com/q/36706429/155423)
- [Why can I return a reference to a local literal but not a variable?](https://stackoverflow.com/q/50345139/155423)

# Your actual problem

Look at the documentation for [`OpenOptions::open`](https://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.open):

```rust
fn open<P: AsRef<Path>>(&self, path: P) -> Result<File>
```

It returns a `Result<File>`, so I don't know how you'd expect to return an `OpenOptions` or a reference to one. Your function would work if you rewrote it as:

```rust
fn trycreate() -> File {
    OpenOptions::new()
        .write(true)
        .open("foo.txt")
        .expect("Couldn't open")
}
```

This uses [`Result::expect`](https://doc.rust-lang.org/std/result/enum.Result.html#method.expect) to panic with a useful error message. Of course, panicking in the guts of your program isn't super useful, so it's recommended to propagate your errors back out:

```rust
fn trycreate() -> io::Result<File> {
    OpenOptions::new().write(true).open("foo.txt")
}
```

`Option` and `Result` have lots of nice methods to deal with chained error logic. Here, you can use [`or_else`](https://doc.rust-lang.org/std/result/enum.Result.html#method.or_else):

```rust
let f = OpenOptions::new().write(true).open("foo.txt");
let mut f = f.or_else(|_| trycreate()).expect("failed at creating");
```

I'd also return the `Result` from `main`. All together, including fjh's suggestions:

```rust
use std::{
    fs::OpenOptions,
    io::{self, Write},
};

fn main() -> io::Result<()> {
    let mut f = OpenOptions::new()
        .create(true)
        .write(true)
        .append(true)
        .open("foo.txt")?;

    f.write_all(b"test1\n")?;
    f.write_all(b"test2\n")?;

    Ok(())
}
```

answer2

> Is there any way to return a reference from a function without arguments?

No (except references to static values, but those aren't helpful here).

However, you might want to look at [`OpenOptions::create`](http://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.create). If you change your first line in `main` to

```rust
let  f = OpenOptions::new().write(true).create(true).open(b"foo.txt");
```

the file will be created if it does not yet exist, which should solve your original problem.

answer3

References are pointers. Once functions are executed, they are popped off the execution stack and resources are de-allocated.

For the following example, `x` is dropped at the end of the block. After that point, the reference `&x` will be pointing to some garbage data. Basically it is a dangling pointer. The Rust compiler does not permit such a thing since it is not safe.

```rust
fn run() -> &u32 {
    let x: u32 = 42;

    return &x;
} // x is dropped here

fn main() {
    let x = run();
}
```

answer4

Rust doesn't allow return a reference to a variable created in a function. Is there a workaround? Yes, simply put that variable in a [Box](https://doc.rust-lang.org/std/boxed/struct.Box.html) then return it. Example:

```rust
fn run() -> Box<u32> {
    let x: u32 = 42;
    return Box::new(x);
} 

fn main() {
    println!("{}", run());
}
```

[code in rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=a4fbb88cfbdd5e5459b883895d043d63)

As a rule of thumb, to avoid similar problems in Rust, return an owned object (Box, Vec, String, ...) instead of reference to a variable:

- `Box<T>` instead of `&T`
- `Vec<T>` instead of `&[T]`
- `String` instead of `&str`

For other types, refer to [The Periodic Table of Rust Types](http://cosmic.mearie.org/2014/01/periodic-table-of-rust-types/) to figure out which owned object to use.

Of course, in this example you can simply return the value (`T` instead of `&T` or `Box<T>`)

```rust
fn run() -> u32 {
    let x: u32 = 42;
    return x;
} 
```
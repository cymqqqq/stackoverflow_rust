How to hold an iterator inside a struct in rust

I'm trying to do this

```rust
struct RwindIter {
    iter: Box<dyn Iterator<Item = String>>,
}

fn build(i: impl Iterator<Item = String>) -> RwindIter {
    RwindIter { iter: Box::new(i) }
}
```

But I got this error

```rust
   Compiling myml v0.1.0 (/Users/gecko/code/myml)
error[E0310]: the parameter type `impl Iterator<Item = String>` may not live long enough
  --> src/main.rs:47:23
   |
47 |     RwindIter { iter: Box::new(i) }
   |                       ^^^^^^^^^^^
   |
note: ...so that the type `impl Iterator<Item = String>` will meet its required lifetime bounds
  --> src/main.rs:47:23
   |
47 |     RwindIter { iter: Box::new(i) }
   |                       ^^^^^^^^^^^
help: consider adding an explicit lifetime bound  `'static` to `impl Iterator<Item = String>`...
   |
46 | fn build(i: impl Iterator<Item = String> + 'static) -> RwindIter {
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0310`.
error: could not compile `myml`.

To learn more, run the command again with --verbose.
```

I was expecting that `Box::new(x)` would take the onwership of `x` so I can't figure out what the error message means. Any ideas?

------

Update

Okay I know it's some limitation around `impl` syntax. This works

```rust
struct I {}

impl Iterator for I {
    type Item = String;
    fn next(&mut self) -> Option<String> {
        None
    }
}

fn build(i: I) -> RwindIter {
    RwindIter { iter: Box::new(i) }
}
```

answer1

I would recommend just using a regular generic to solve the issue.

```rust
pub struct FooIter<I> {
    iter: I,
}

impl<I> FooIter<I> {
    pub fn new(iter: I) -> Self {
        FooIter { iter }
    }
}

impl<I: Iterator<Item=String>> Iterator for FooIter<I> {
    type Item = String;
    
    fn next(&mut self) -> Option<Self::Item> {
        self.iter.next()
    }
}
```

However you can still use a `dyn Iterator<Item=String>` so long as you provide a lifetime bound. However it could lead to a massive mess of lifetimes later on during implementation depending on how you interact with this struct.

```rust
pub struct FooIter<'a> {
    iter: Box<dyn Iterator<Item=String> + 'a>,
}

impl<'a> FooIter<'a> {
    pub fn new(iter: impl Iterator<Item=String> + 'a) -> Self {
        FooIter {
            iter: Box::new(iter),
        }
    }
}

impl<'a> Iterator for FooIter<'a> {
    type Item = String;
    
    fn next(&mut self) -> Option<Self::Item> {
        self.iter.next()
    }
}
```


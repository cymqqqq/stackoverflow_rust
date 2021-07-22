How to define an iterator in Rust over a struct that contains items that are iterable?

How does one define an iterator in Rust over a struct that contains items that are already iterable? Here's one attempt at the iterator

```rust
use rand;

// Structure of items
struct Foo {
    foo: Vec<f64>,
    bar: Vec<i64>,
}

// Iterator for the structure
struct FooIter {
    foo: Iterator,
    bar: Iterator,
}

// Method that provides the iterator for use
impl Foo {
    fn iter(&self) -> FooIter {
        FooIter {
            foo: self.foo.iter().peek(),
            bar: self.bar.iter().peek(),
        }
    }
}

// Item desired from iterator
enum Bar {
    MyFloat(f64),
    MyInt(i64),
}

// Implementation of the iterator
impl Iterator for FooIter {
    type Item = Bar;

    fn next(&mut self) -> Option<Bar> {
        match (self.foo.peek(), self.far.peek()) {
            (Some(_), Some(_)) => {
                if rand::random() {
                    self.foo.next()
                } else {
                    self.bar.next()
                }
            }
            (Some(_), None) => self.foo.next(),
            (None, Some(_)) => self.bar.next(),
            (None, None) => None,
        }
    }
}

// Iterate over a struct
fn main() {
    let fuz = Foo {
        foo: vec![1.2, 2.3, 3.4],
        bar: vec![5, 6],
    };
    for item in fuz.iter() {
        match item {
            Bar::MyFloat(f) => println!("float : {}", f),
            Bar::MyInt(i) => println!("int : {}", i),
        }
    }
}
```

In short, the struct `Foo` contains two vectors and I'd like an iterator that jumps back and forth between the two elements randomly. Certainly, there are many mistakes here, but at the core, I don't understand how to create a struct that carries the iterators for the items `foo` and `far` because Rust defines iterators as a trait and not a type.

answer1

You must at some point define what `Item` the `Iterator` will produce, for example `Iterator<Item = &'a f64>`. Let simplify and transform to `Iterator<Item = f64>`, because `f64` is `Copy` so it's often better to avoid the reference if you don't need it.

So, then we will have compile error:

```none
error[E0277]: the size for values of type `(dyn std::iter::Iterator<Item = f64> + 'static)` cannot be known at compilation time
  --> src/main.rs:11:5
   |
11 |     foo: std::iter::Iterator<Item = f64>,
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ doesn't have a size known at compile-time
   |
   = help: the trait `std::marker::Sized` is not implemented for `(dyn std::iter::Iterator<Item = f64> + 'static)`
   = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
   = note: only the last field of a struct may have a dynamically sized type
```

To avoid dynamic type and fix the error at the same time, let's define some generic type:

```rust
// Iterator for the structure
struct FooIter<F, I> {
    foo: F,
    bar: I,
}
```

We add the necessary on our implementation of `Iterator`:

```rust
impl<F, I> Iterator for FooIter<F, I>
where
    F: Iterator<Item = f64>,
    I: Iterator<Item = i64>,
```

And we must change how we generate `FooIter`, this time we will use a magic keyword `impl`, this avoid to write the real type of the `Iterator` that can be very long and unclear, the compiler will infer the type for us. Also, we must bound the type to the lifetime of `&self` because it must be borrow as long as the iterator live, simply declare `'a` lifetime and add `+ 'a` will do the job:

```rust
fn iter<'a>(
    &'a self,
) -> FooIter<impl Iterator<Item = f64> + 'a, impl Iterator<Item = i64> + 'a> {
    FooIter {
        foo: self.foo.iter().copied(),
        bar: self.bar.iter().copied(),
    }
}
```

Here we finish the basic, the next problem is that your code doesn't produce `Bar` type in `next()`, so we must correct your code, also it's would be nice to create a propre random generator. So here the final snippet:

```rust
use rand::{rngs::ThreadRng, thread_rng, Rng};

// Structure of items
struct Foo {
    foo: Vec<f64>,
    bar: Vec<i64>,
}

// Iterator for the structure
struct FooIter<'r, F, I> {
    foo: F,
    bar: I,
    rng: &'r mut ThreadRng,
}

// Method that provides the iterator for use
impl Foo {
    fn iter<'a, 'r: 'a>(
        &'a self,
        rng: &'r mut ThreadRng,
    ) -> FooIter<impl Iterator<Item = f64> + 'a, impl Iterator<Item = i64> + 'a> {
        FooIter {
            foo: self.foo.iter().copied(), // nigthly feature, use cloned() for stable
            bar: self.bar.iter().copied(),
            rng,
        }
    }
}

// Item desired from iterator
enum Bar {
    MyFloat(f64),
    MyInt(i64),
}

// Implementation of the iterator
impl<'r, F, I> Iterator for FooIter<'r, F, I>
where
    F: Iterator<Item = f64>,
    I: Iterator<Item = i64>,
{
    type Item = Bar;

    fn next(&mut self) -> Option<Bar> {
        if self.rng.gen() {
            self.foo
                .next()
                .map(|x| Bar::MyFloat(x))
                .or_else(|| self.bar.next().map(|x| Bar::MyInt(x)))
        } else {
            self.bar
                .next()
                .map(|x| Bar::MyInt(x))
                .or_else(|| self.foo.next().map(|x| Bar::MyFloat(x)))
        }
    }
}

// Iterate over a struct
fn main() {
    let fuz = Foo {
        foo: vec![1.2, 2.3, 3.4],
        bar: vec![5, 6],
    };
    for item in fuz.iter(&mut thread_rng()) {
        match item {
            Bar::MyFloat(f) => println!("float : {}", f),
            Bar::MyInt(i) => println!("int : {}", i),
        }
    }
}
```

------

Note, if you still want a `Peekable<Iterator>` then just do:

```rust
struct FooIter<'r, F, I>
where
    F: Iterator<Item = f64>,
    I: Iterator<Item = i64>,
{
    foo: Peekable<F>,
    bar: Peekable<I>,
    rng: &'r mut ThreadRng,
}

// Method that provides the iterator for use
impl Foo {
    fn iter<'a, 'r: 'a>(
        &'a self,
        rng: &'r mut ThreadRng,
    ) -> FooIter<impl Iterator<Item = f64> + 'a, impl Iterator<Item = i64> + 'a> {
        FooIter {
            foo: self.foo.iter().copied().peekable(),
            bar: self.bar.iter().copied().peekable(),
            rng,
        }
    }
}
```

answer2

I wanted to include two separate codes to complete things. The following is the fixed code that's slightly closer to my original attempt, which works on Rust 1.34.1:

```rust
// Structure of items
struct Foo {
    foo: Vec<f64>,
    far: Vec<i64>,
}

// Iterator for the structure
struct FooIter<FloatIter, IntIter>
where
    FloatIter: Iterator<Item = f64>,
    IntIter: Iterator<Item = i64>,
{
    foo: std::iter::Peekable<FloatIter>,
    far: std::iter::Peekable<IntIter>,
}

// Method that provides the iterator for use
impl Foo {
    fn iter<'a>(
        &'a self,
    ) -> FooIter<impl Iterator<Item = f64> + 'a, impl Iterator<Item = i64> + 'a> {
        FooIter {
            foo: self.foo.iter().cloned().peekable(),
            far: self.far.iter().cloned().peekable(),
        }
    }
}

// Item desired from iterator
enum Bar {
    MyFloat(f64),
    MyInt(i64),
}

// Implementation of the iterator
impl<FloatIter, IntIter> Iterator for FooIter<FloatIter, IntIter>
where
    FloatIter: Iterator<Item = f64>,
    IntIter: Iterator<Item = i64>,
{
    type Item = Bar;

    fn next(&mut self) -> Option<Bar> {
        match (self.foo.peek(), self.far.peek()) {
            (Some(_), Some(_)) => {
                if rand::random() {
                    self.foo.next().map(|x| Bar::MyFloat(x))
                } else {
                    self.far.next().map(|x| Bar::MyInt(x))
                }
            }
            (Some(_), None) => self.foo.next().map(|x| Bar::MyFloat(x)),
            (None, Some(_)) => self.far.next().map(|x| Bar::MyInt(x)),
            (None, None) => None,
        }
    }
}

// Iterate over a struct
fn main() {
    let fuz = Foo {
        foo: vec![1.2, 2.3, 3.4],
        far: vec![5, 6],
    };
    for item in fuz.iter() {
        match item {
            Bar::MyFloat(f) => println!("float : {}", f),
            Bar::MyInt(i) => println!("int : {}", i),
        }
    }
}
```

What helped me understand what was going on is that `FooIter` parametrizes its arguments on generic types. These types are inferred by using `impl Trait` in the return position within the `iter` method for `Foo`. That said, I was able to write a similar code without using this inference:

```rust
extern crate rand;

// Structure of items
struct Foo {
    foo: Vec<f64>,
    far: Vec<i64>,
}

// Iterator for the structure
struct FooIter<'a> {
    foo: std::iter::Peekable<std::slice::Iter<'a, f64>>,
    far: std::iter::Peekable<std::slice::Iter<'a, i64>>,
}

// Method that provides the iterator for use
impl Foo {
    fn iter<'a>(&'a self) -> FooIter<'a> {
        FooIter {
            foo: self.foo.iter().peekable(),
            far: self.far.iter().peekable(),
        }
    }
}

// Item desired from iterator
enum Bar {
    MyFloat(f64),
    MyInt(i64),
}

// Implementation of the iterator
impl<'a> Iterator for FooIter<'a> {
    type Item = Bar;

    fn next(&mut self) -> Option<Bar> {
        match (self.foo.peek(), self.far.peek()) {
            (Some(_), Some(_)) => {
                if rand::random() {
                    self.foo.next().map(|x| Bar::MyFloat(x.clone()))
                } else {
                    self.far.next().map(|x| Bar::MyInt(x.clone()))
                }
            }
            (Some(_), None) => self.foo.next().map(|x| Bar::MyFloat(x.clone())),
            (None, Some(_)) => self.far.next().map(|x| Bar::MyInt(x.clone())),
            (None, None) => None,
        }
    }
}

// Iterate over a struct
fn main() {
    let fuz = Foo {
        foo: vec![1.2, 2.3, 3.4],
        far: vec![5, 6],
    };
    for item in fuz.iter() {
        match item {
            Bar::MyFloat(f) => println!("float : {}", f),
            Bar::MyInt(i) => println!("int : {}", i),
        }
    }
}
```

This is almost certainly the wrong way to do things, but I wanted to see if it was possible. I determined the iterator type by compiling the code:

```rust
fn main() {
    let x = vec![1.2, 2.3, 3.4];
    let y: i32 = x.iter().peekable();
}
```

which gave the compiler error:

```none
error[E0308]: mismatched types
 --> junk.rs:4:19
  |
4 |     let y: i32 = x.iter().peekable();
  |                  ^^^^^^^^^^^^^^^^^^^ expected i32, found struct `std::iter::Peekable`
  |
  = note: expected type `i32`
             found type `std::iter::Peekable<std::slice::Iter<'_, {float}>>`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0308`.
```

This contains the type of what I was looking for. Again, this is almost certainly the wrong thing to do, but it helped me understand the provided answer.


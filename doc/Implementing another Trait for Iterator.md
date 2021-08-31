Implementing another Trait for Iterator

I'm trying to add another trait to all `Iterator`s. But I don't understand why it doesn't compile.

Here is the code:

```rust
use std::fmt::Display;
use std::iter::Sum;

trait Topla<T> {
    fn topla(&mut self)-> T;
}

impl<T, I> Topla<T> for I
where
    T: Sum + Display,
    I: Iterator<Item = T>,
{
    fn topla(&mut self) -> T {
        self.sum()
    }
}

fn main() {
    let data = vec![1,2,3,5,8];
    println!("{:?}", data.iter().topla());
}
```

answer

The problem reveals itself if we use fully-qualified trait syntax:

```rs
fn main() {
    let data = vec![1,2,3,5,8u32];
    let mut iter = data.iter();
    println!("{:?}", Topla::<u32>::topla(&mut iter));
}
error[E0271]: type mismatch resolving `<std::slice::Iter<'_, u32> as std::iter::Iterator>::Item == u32`
  --> src/main.rs:22:22
   |
5  |     fn topla(&mut self) -> T;
   |     ------------------------- required by `Topla::topla`
...
22 |     println!("{:?}", Topla::<u32>::topla(&mut iter));
   |                      ^^^^^^^^^^^^^^^^^^^ expected reference, found `u32`
   |
   = note: expected reference `&u32`
                   found type `u32`
   = note: required because of the requirements on the impl of `Topla<u32>` for `std::slice::Iter<'_, u32>`
```

Changing `Topla<u32>` to `Topla<&u32>` gets us closer:

```none
Compiling playground v0.0.1 (/playground)
error[E0277]: the trait bound `&u32: std::iter::Sum` is not satisfied
  --> src/main.rs:22:22
   |
5  |     fn topla(&mut self) -> T;
   |     ------------------------- required by `Topla::topla`
...
22 |     println!("{:?}", Topla::<&u32>::topla(&mut iter));
   |                      ^^^^^^^^^^^^^^^^^^^^ the trait `std::iter::Sum` is not implemented for `&u32`
   |
   = help: the following implementations were found:
             <u32 as std::iter::Sum<&'a u32>>
             <u32 as std::iter::Sum>
   = note: required because of the requirements on the impl of `Topla<&u32>` for `std::slice::Iter<'_, u32>`
```

The problem is that `Sum<&u32>` isn't implemented for `&u32`; it's implemented for `u32`. Since we need to return a `u32` and not `&u32`, we need to loosen our requirements on `T`, from being the iterator type itself to just being `Sum`able with T.

```rs
trait Topla<T> {
    fn topla(&mut self) -> T;
}

impl<T, I> Topla<T> for I
where
    T: Display + Sum<<I as Iterator>::Item>,
    I: Iterator,
{
    fn topla(&mut self) -> T {
        self.sum()
    }
}

fn main() {
    let data = vec![1,2,3,5,8u32];
    let mut iter = data.iter();
    println!("{:?}", Topla::<u32>::topla(&mut iter));
}
```

But now if we return to the original syntax, we get type inference failures that make our new API very annoying to use. We can get around this by being a little more strict with the API.

If we make `Topla` a subtrait of `Iterator`, we can reference the item type in the definition of the `topla` method, and therefore move the type parameter for the output into the method rather than the trait. This will let us use the turbofish syntax like we do with `sum()`. Finally we have:

```rs
use std::fmt::Display;
use std::iter::Sum;

trait Topla: Iterator {
    fn topla<T>(self) -> T
    where
        Self: Sized,
        T: Sum<<Self as Iterator>::Item>;
}

impl<I> Topla for I
where
    I: Iterator,
    <I as Iterator>::Item: Display,
{
    fn topla<T>(self) -> T
    where
        Self: Sized,
        T: Sum<<Self as Iterator>::Item>,
    {
        self.sum()
    }
}

fn main() {
    let data = vec![1, 2, 3, 5, 8];
    println!("{:?}", data.iter().topla::<u32>());
}
```


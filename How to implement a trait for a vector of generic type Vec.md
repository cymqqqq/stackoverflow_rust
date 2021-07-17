How to implement a trait for a vector of generic type Vec

How to implement the below trait for a vector of generic type `Vec<T>`?

For example, how to implement the below (working) `Difference` trait in a generic way (e.g. so that it is valid for `Vec<i32>`, `Vec<f32>`, `Vec<f64>`)?

```rust
trait Difference {
    fn diff(&self) -> Vec<f64>;
}

impl Difference for Vec<f64> {
    fn diff(&self) -> Vec<f64> {
        self.windows(2)
            .map(|slice| (slice[0] - slice[1]))
            .collect()
    }
}

fn main() {
    let vector = vec![1.025_f64, 1.028, 1.03, 1.05, 1.051];
    println!("{:?}", vector.diff());
}
```

From looking [at the documentation](https://doc.rust-lang.org/book/second-edition/ch19-03-advanced-traits.html#associated-types-versus-generics), it seems like it should be something along the lines of:

```rust
trait Difference<Vec<T>> {
    fn diff(&self) -> Vec<T>;
}

impl Difference for Vec<T> {
    fn diff(&self) -> Vec<T> {
        self.windows(2)
            .map(|slice| (slice[0] - slice[1]))
            .collect()
    }
}

fn main() {
    let vector = vec![1.025_f64, 1.028, 1.03, 1.05, 1.051];
    println!("{:?}", vector.diff());
}
```

However the above results in:

```none
error: expected one of `,`, `:`, `=`, or `>`, found `<`
 --> src/main.rs:2:21
  |
2 | trait Difference<Vec<T>> {
  |                     ^ expected one of `,`, `:`, `=`, or `>` here
```

I've tried a few other variations however all of them resulted in much longer error messages.

Answer 1

The correct syntax is:

```rust
trait Difference<T> { /* ... */ }

impl<T> Difference<T> for Vec<T> { /* ... */ }
```

Then you will need to require that `T` implements subtraction:

```none
error[E0369]: binary operation `-` cannot be applied to type `T`
 --> src/main.rs:9:26
  |
9 |             .map(|slice| (slice[0] - slice[1]))
  |                          ^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `T` might need a bound for `std::ops::Sub`
```

And that you can copy the values:

```none
error[E0508]: cannot move out of type `[T]`, a non-copy slice
  --> src/main.rs:10:27
   |
10 |             .map(|slice| (slice[0] - slice[1]))
   |                           ^^^^^^^^ cannot move out of here
impl<T> Difference<T> for Vec<T>
where
    T: std::ops::Sub<Output = T> + Copy,
{
    // ...
}
```

Or that references to `T` can be subtracted:

```rust
impl<T> Difference<T> for Vec<T>
where
    for<'a> &'a T: std::ops::Sub<Output = T>,
{
    fn diff(&self) -> Vec<T> {
        self.windows(2)
            .map(|slice| &slice[0] - &slice[1])
            .collect()
    }
}
```

Answer2 

You need to parameterise over `T` not `Vec<T>`. Then you'll also need to constrain `T` so that you can do subtraction (with the `Sub` trait) and so that the values can be copied in memory (with the `Copy` trait). Numeric types will mostly implement these traits.

```rust
use std::ops::Sub;

trait Difference<T> {
    fn diff(&self) -> Vec<T>;
}

impl<T> Difference<T> for Vec<T>
where
    T: Sub<Output = T> + Copy,
{
    fn diff(&self) -> Vec<T> {
        self.windows(2).map(|slice| slice[0] - slice[1]).collect()
    }
}

fn main() {
    let vector = vec![1.025_f64, 1.028, 1.03, 1.05, 1.051];
    println!("{:?}", vector.diff());
}
```


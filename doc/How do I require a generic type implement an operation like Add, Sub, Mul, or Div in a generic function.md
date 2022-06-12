How do I require a generic type implement an operation like Add, Sub, Mul, or Div in a generic function

I'm trying to implement a generic function in Rust where the only requirement for the argument is that the multiplication operation should be defined. I'm trying to implement a generic "power", but will go with a simpler `cube` function to illustrate the problem:

```rust
use std::ops::Mul;

fn cube<T: Mul>(x: T) -> T {
    x * x * x
}

fn main() {
    println!("5^3 = {}", cube(5));
}
```

When compiling I get this error:

```none
error[E0369]: binary operation `*` cannot be applied to type `<T as std::ops::Mul>::Output`
 --> src/main.rs:4:5
  |
4 |     x * x * x
  |     ^^^^^^^^^
  |
  = note: an implementation of `std::ops::Mul` might be missing for `<T as std::ops::Mul>::Output`
```

What does this mean? Did I choose the wrong trait? How can I resolve this?

answer1

Let's break down your example a bit:

```rust
fn cube<T: Mul>(x: T) -> T {
    let a = x * x;
    let b = a * x;
    b
}
```

What are the types of `a` and `b`? In this case, the type of `a` is `<T as std::ops::Mul>::Output` — sound familiar from the error message? Then, we are trying to multiply that type by `x` again, but there's no guarantee that `Output` is able to be multiplied by anything!

Let's do the simplest thing and say that `T * T` needs to result in a `T`:

```rust
fn cube<T: Mul<Output = T>>(x: T) -> T {
    x * x * x
}
```

Unfortunately, this gives two similar errors:

```none
error[E0382]: use of moved value: `x`
 --> src/lib.rs:6:9
  |
6 |     x * x * x
  |     -   ^ value used here after move
  |     |
  |     value moved here
  |
  = note: move occurs because `x` has type `T`, which does not implement the `Copy` trait
```

Which is because the [`Mul` trait takes arguments by value](http://doc.rust-lang.org/std/ops/trait.Mul.html), so we add the `Copy` so we can duplicate the values.

I also switched to the `where` clause as I like it better and it is unwieldy to have that much inline:

```rust
fn cube<T>(x: T) -> T
where
    T: Mul<Output = T> + Copy
{
    x * x * x
}
```

answer2

The bound `T: Mul` does not imply that the result of the binary operator is also of type `T`. The result type is an *associated type* of this trait: `Output`.

The other issue is that before Rust 1.0 the operator traits switched from pass-by-reference to pass-by-value. In generic code this can be a bit of a pain in the butt (for now at least) because these operators *consume* their operands unless you also require the types to be `Copy`.

Just for completeness (in case you don't like to require `Copy`), let me add some information about a possible alternative direction.

For the sake of generic code, authors of "numeric types" are encouraged to provide additional *non-consuming* implementations of these operator traits so that you don't need `Copy` or `Clone`. For example, the standard library already provides the following implementations:

```none
 f64 implements Mul< f64>
 f64 implements Mul<&f64>
&f64 implements Mul< f64>
&f64 implements Mul<&f64>
```

Each of these implementations has `f64` as the `Output` type. Making use of these traits directly is not pretty:

```rust
fn cube<T>(x: &T) -> T
where
    for<'a> T: Mul<&'a T, Output = T>,
    for<'a, 'b> &'a T: Mul<&'b T, Output = T>,
{
    x * x * x
}
```

Eventually, we might get some (slightly) higher level traits, which would reduce the noise. For example: `T: Mul2` could imply `T: Mul<T> + Mul<&T>` and `&T: Mul<T> + Mul<&T>`, but at the time of writing this, the Rust compiler does not seem able to handle this. At least I could not successfully compile the following code:

```rust
use std::ops::Mul;

pub trait Mul2
where
    Self: Mul<Self, Output = Self>,
    Self: for<'a> Mul<&'a Self, Output = Self>,
    for<'a> &'a Self: Mul<Self, Output = Self>,
    for<'a, 'b> &'a Self: Mul<&'b Self, Output = Self>,
{
}

impl<T> Mul2 for T
where
    T: Mul<T, Output = T>,
    T: for<'a> Mul<&'a T, Output = T>,
    for<'a> &'a T: Mul<T, Output = T>,
    for<'a, 'b> &'a T: Mul<&'b T, Output = T>,
{
}

fn cube<T: Mul2>(x: &T) -> T {
    x * x * x
}

fn main() {
    let c = cube(&2.3);
    println!("Hello, world! {}", c)
}
```


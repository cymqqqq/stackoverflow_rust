Using a generic in a struct and implementing via a trait

I have struct defined like this

```rust
struct Shape<T> {
    line_one: T,
    line_two: T,
}
```

I am trying to create a simple trait and implementation that takes this struct to calculate some simple math.

My trait and impls look like this

```rust
trait Calculate<T: Mul<Output = T>> {
    fn calc(&self, Shape: T) -> T;
}

impl<T> Calculate<T> for Shape<T> {
    fn calc(&self, Shape: T) -> T {
        self.line_one * 2 + self.line_two * 2
    }
}

impl Calculate<i32> {
    fn calc(&self) -> i32 {
        self.line_one * 2 + self.line_two * 2
    }
}

fn calc_fn<T: Calculate<i32>>(calculate: T) {
    calculate.calc();
}
```

When I put this into the Rust playground, the compile fails as `Mul` is not implemented in the `impl Calculate<T>`. However, if I change `<T>` to `<T: Mul<Output = T>>`, I get the error that

```rust
<anon>:14:21: 14:22 error: expected one of `!`, `(`, `+`, `,`, `::`, `<`, or `>`, found `:`
<anon>:14 impl <T> Calculate<T: Mul<Output = T>> for Shape<T> 
```

I'm at a loss as how to implement `Mul` for `T` in `impl Calculate<T>`.

What I'm trying to achieve is code that I can send either a `f32` or `i32` in without needing to create two different impl definitions so can just pass in `T`.

Answer 1

This syntax is incorrect:

```rust
impl<T> Calculate<T: Mul<Output = T>> for Shape<T>
```

you want

```rust
impl<T: Mul<Output = T>> Calculate<T> for Shape<T>
```

I'd almost always recommend using `where` clause instead; I think it reads better and might have helped prevent this case:

```rust
impl<T> Calculate<T> for Shape<T>
    where T: Mul<Output = T>,
```

This just unlocks more errors. You are trying to multiply by `2`, an integral variable that hasn't been fully nailed down yet — is it a `u8`? is it an `i32`? The restrictions don't say that `T` can be multiplied by a `2`. Additionally, you try to add values together, but there's no guarantee that you can add the type `T`.

The `impl Calculate<i32>` and `calc_fn` blocks don't really make sense; it's unlikely you want to [implement functions for the trait](https://stackoverflow.com/q/34438755/155423) and you don't supply a `Shape` in the latter. The trait also doesn't use the `Shape: T` parameter (and Rust uses `snake_case` variable names anyway).

> What I'm trying to achieve is code that I can send either a `f32` or `i32` in without needing to create two different impl definitions so can just pass in `T`.

In this case, it'd probably be way easier:

```rust
trait Calculate<T> {
    fn calc(&self) -> T;
}

impl Calculate<i32> for Shape<i32> {
    fn calc(&self) -> i32 {
        self.line_one * 2 + self.line_two * 2
    }
}

impl Calculate<f32> for Shape<f32> {
    fn calc(&self) -> f32 {
        self.line_one * 2.0 + self.line_two * 2.0
    }
}
```

Note that these are **not** equivalent! The `f64` has floating point literals (`2.0`) not integral (`2`).

If you *must* have a generic, you are going to need to convert the literal `2` to the type `T` or add a restriction that T can be multiplied by some known size. Unfortunately, `f64` cannot by multiplied by any integral value without casting.

The [`FromPrimitive`](http://rust-num.github.io/num/num/trait.FromPrimitive.html) trait from the num crate is key here. Making the types `Copy` also makes the implementation easier.

```rust
extern crate num;

use std::ops::{Add, Mul};
use num::FromPrimitive;

struct Shape<T> {
    line_one: T,
    line_two: T,
}

trait Calculate<T> {
    fn calc(&self) -> T;
}

impl<T> Calculate<T> for Shape<T>
    where T: Copy + FromPrimitive + Add<Output = T> + Mul<Output = T>,
{
    fn calc(&self) -> T {
        let two = T::from_u8(2).expect("Unable to create a value of two");
        self.line_one * two + self.line_two * two
    }
}

fn main() {
    let s1 = Shape { line_one: 2, line_two: 3 };
    let s2 = Shape { line_one: 2.0, line_two: 3.0 };

    println!("{}", s1.calc());
    println!("{}", s2.calc());
}
```
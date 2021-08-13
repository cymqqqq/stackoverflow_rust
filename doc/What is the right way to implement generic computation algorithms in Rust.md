What is the right way to implement generic computation algorithms in Rust?

It is quite a nuisance to implement a generic computation algorithm in Rust. It feels like I am reinventing all the stuff not in the algorithm, but in the codomain of Church numerals.

For example, here's an implementation of `factorial` that works in Rust 1.7:

```rust
#![feature(zero_one)]

use std::num::{One, Zero};
use std::ops::{Sub, Mul};
use std::cmp::Eq;

fn fact<T>(n: T) -> T
    where T: Clone + Eq + Zero + One + Mul<T, Output = T> + Sub<T, Output = T>
{
    if n == T::zero() {
        T::one()
    } else {
        fact(n.clone() - T::one()) * n
    }
}

fn main() {
    println!("{}", fact(10));
}
```

Is there any right way of doing this? Is there any discussion going on with it?

------

Probably `factorial` is not good example, let's try `is_even`:

```rust
fn is_even<T>(x: T) -> bool
    where T: std::ops::Rem<Output = T> + std::ops::Add<T, Output=T> + std::num::One + std::num::Zero + std::cmp::PartialEq
{
    let two = T::one() + T::one();
    (x % two) == T::zero()
}
```

If you want a `two` stuff, you must reimplement two.

answer

If I wanted to implement `is_even` I would obviously start by implementing `is_divisible` which is more generic:

```rust
#![feature(zero_one)]

use std::cmp;
use std::num;
use std::ops;

fn is_divisible<T>(x: T, by: T) -> bool
    where T: ops::Rem<Output = T> + num::Zero + cmp::PartialEq
{
    (x % by) == T::zero()
}
```

It seems easy enough.

However, `is_even` has even more constraints and this is getting a bit long, so let's follow DRY:

```rust
trait Arithmetic:
    From<u8> +
    cmp::PartialEq + cmp::Eq + cmp::PartialOrd + cmp::Ord +
    ops::Add<Self, Output = Self> + ops::Sub<Self, Output = Self> +
    ops::Mul<Self, Output = Self> + ops::Div<Self, Output = Self> + ops::Rem<Self, Output = Self> {}

impl<T> Arithmetic for T
    where T: From<u8> +
             cmp::PartialEq + cmp::Eq + cmp::PartialOrd + cmp::Ord +
             ops::Add<T, Output = T> + ops::Sub<T, Output = T> +
             ops::Mul<T, Output = T> + ops::Div<T, Output = T> + ops::Rem<T, Output = T>
 {}
```

Alright, this trait should cover us. Mind that it's missing a `ops::Neg` bound because this bound is not implemented for unsigned traits; so if we need `Neg` we'll have to add it; but it's easy enough.

As for the issue about constants, well indeed working your way from `zero` upwards is insane. It's quite the reason why the `Zero` and `One` traits are still unstable.

The generic conversion traits are `convert::From` and `convert::Into`, and that is what one would use.

So let us reformulate `is_divisible`, and finally implement `is_even`:

```rust
fn is_divisible<T>(x: T, by: T) -> bool
    where T: Arithmetic
{
    (x % by) == 0.into()
}

fn is_even<T>(x: T) -> bool
    where T: Arithmetic
{
    is_divisible(x, 2.into())
}
```

And really, those two functions seem both perfectly clear whilst still being generic.

[Full code here](https://play.rust-lang.org/?gist=cf63456afd343144c7e6&version=stable)

------

Now, we might argue that creating this `Arithmetic` trait is a long-winded way of getting to `is_even`. It is. However:

- if you only need `is_even`, obviously you care little if it takes 6 bounds; it's a one off
- if you need multiple generic functions working on numerics, then the small cost of creating this trait and function are negligible in the grand scheme of things

In short, it works. And it's really not that onerous.
How to return an Iterator with associated type being generic from a function?

For a function like this:

```rust
fn generate_even(a: i32, b: i32) -> impl Iterator<Item = i32> {
    (a..b).filter(|x| x % 2 == 0)
}
```

I want to make it generic, instead of the concrete type `i32` I want to have any type that is range-able and provide a filter implementation.

Tried the formula below without success:

```rust
fn generate_even(a: T, b: T) -> impl Iterator<Item = T>
    where T: // range + filter + what to put here?
{
    (a..b).filter(|x| x % 2 == 0)
}
```

How can something like this be implemented?

answer1

```rust
use std::ops::Range;
use std::ops::Rem;
use std::cmp::PartialEq;

fn generate_even<T>(a: T, b: T) -> impl Iterator<Item = T>
  where
    Range<T>: Iterator<Item = T>,
    T: Copy + Rem<Output = T> + From<u8> + PartialEq
{
    let zero: T = 0_u8.into();
    let two: T = 2_u8.into();
    (a..b).filter(move |&x| x % two == zero)
}

fn main() {
    let even_u8s = generate_even(0_u8, 11_u8);
    let even_i16s = generate_even(0_i16, 11_i16);
    let even_u16s = generate_even(0_u16, 11_u16);
    // and so on
    let even_u128s = generate_even(0_u128, 11_u128);
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=7d8e1b9d1fbe3de39ab097b5de0dee62)

The hardest part of the solution is implementing `x % 2 == 0` in a generic way because by default Rust interprets integer literals as `i32`s but you want your function to be generic across *all* possible integer types, which means you have to produce a `2` and `0` value of whatever integer type the caller specifies, and the simplest way to do that is to bound `T` by `From<u8>` which allows us to transform any value from `0` to `256` into any integer type (with `i8` being the only exception). The above solution is generic and works for all integer types except `i8`.

answer2

It's actually possible to support all integer types, including `i8`, by using `TryInto`.

```rust
use std::ops::Range;
use std::ops::Rem;
use std::cmp::PartialEq;
use std::fmt;
use std::convert::{TryFrom, TryInto};

fn generate_even<T>(a: T, b: T) -> impl Iterator<Item = T>
  where
    Range<T>: Iterator<Item = T>,
    T: Copy + Rem<Output = T> + TryFrom<u8> + PartialEq + fmt::Debug,
    <T as TryFrom<u8>>::Error: fmt::Debug
{
    let zero: T = 0_u8.try_into().unwrap();
    let two: T = 2_u8.try_into().unwrap();
    (a..b).filter(move |&x| x % two == zero)
}

fn main() {
    let even_u8s = generate_even(0_i8, 11_i8);
    let even_u8s = generate_even(0_u8, 11_u8);
    let even_i16s = generate_even(0_i16, 11_i16);
    let even_u16s = generate_even(0_u16, 11_u16);
    // and so on
    let even_u128s = generate_even(0_u128, 11_u128);
}
```


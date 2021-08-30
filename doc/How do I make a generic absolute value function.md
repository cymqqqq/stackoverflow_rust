How do I make a generic absolute value function?

I'm attempting to write a generic function that calculates the absolute value of any signed integer type. It should return an error when the value is the lowest possible negative value, e.g for 8 bits `abs(-128)` cannot be represented.

I got this working for `i8`:

```rust
pub fn abs(x: i8) -> Result<i8, String> {
    match x {
        x if x == -128i8 => Err("Overflow".to_string()),
        // I know could just use x.abs() now but this illustrates a problem in the generic version below...
        x if x < 0i8 => Ok(-x),
        _ => Ok(x),
    }
}

fn main() {
    println!("{:?}", abs(-127i8));
    println!("{:?}", abs(-128i8));
}
```

I can't get the generic version to work. Specifically I've two problems:

- How do I generically determine the minimum value? What is the Rust equivalent of the C++ `std::numeric_limits<T>::min()`? There is e.g. `std::i32::MIN` but I can't write `std::T::MIN`.
- My generic implementation errors on the match arm for negative values with "cannot bind by-move into a pattern guard" (yet the non-generic version doesn't.)

```rust
use num::{traits::Zero, Integer, Signed}; // 0.2.0

pub fn abs<T>(x: T) -> Result<T, String>
where
    T: Signed + Integer + Zero,
{
    match x {
        //x if x == ***rust equivalent of std::numeric_limits<T>::min()** => Err("Overflow".to_string()),
        x if x < T::zero() => Ok(-x),
        _ => Ok(x),
    }
}

fn main() {
    println!("{:?}", abs(-127i8));
    println!("{:?}", abs(-128i8));
}
error[E0008]: cannot bind by-move into a pattern guard
 --> src/main.rs:9:9
  |
9 |         x if x < T::zero() => Ok(-x),
  |         ^ moves value into pattern guard
```

answer

> How do I determine generically the minimum value? Basically the Rust equivalent of the C++ `std::numeric_limits<T>::min()`?

You want the [`Bounded` trait](https://rust-num.github.io/num/num_traits/bounds/trait.Bounded.html) from the `num-traits` or `num` crates, which gives you a `min_value` method:

```rust
pub fn abs<T>(x: T) -> Result<T, String>
where
    T: Signed + Integer + Zero + Neg + Bounded + Copy,
{
    match x {
        x if x == T::min_value() => Err("Overflow".to_string()),
        x if x < T::zero() => Ok(-x),
        _ => Ok(x),
    }
}
```

> My generic implementation errors on the match arm for negative values with "cannot bind by-move into a pattern guard" (yet the non-generic version doesn't.)

I added a `Copy` bound, to avoid the issue of moving the values in the pattern guard. Most numeric types should be `Copy`.

Perhaps better though would be to use a "checked" operator variant, e.g. `CheckedSub`:

```rust
pub fn abs<T>(x: T) -> Result<T, String>
where
    T: Signed + Integer + Zero + Neg + CheckedSub,
{
    if x < T::zero() {
        T::zero()
            .checked_sub(&x)
            .ok_or_else(|| String::from("Overflow"))
    } else {
        Ok(x)
    }
}
```

This delegates the "meat" of the function to existing code that does exactly what you want, so you have less room for mistakes.


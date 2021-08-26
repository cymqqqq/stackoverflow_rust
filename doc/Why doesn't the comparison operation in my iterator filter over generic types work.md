Why doesn't the comparison operation in my iterator filter over generic types work?

I am trying to understand why the following code does not compile ([Playground](https://play.rust-lang.org/?gist=3e93c5b7784f83bd3c79e295f52e3948&version=stable&mode=debug&edition=2015)):

```rust
fn inspection<I>(iter: I)
where 
    I: Iterator, 
    I::Item: Ord,
{
    let inspection = iter
        .filter(|x| x > 0);
}

fn main() {
    let data = vec![1, 2, 3];
    inspection(data.iter()); // or inspection(data.into_iter());
}
```

The error is:

```none
error[E0308]: mismatched types
 --> src/main.rs:9:25
  |
9 |         .filter(|x| x > 0);
  |                         ^ expected reference, found integral variable
  |
  = note: expected type `&<I as std::iter::Iterator>::Item`
             found type `{integer}`
```

I tried to follow the various alternatives (by de-referencing the element) as explained [here](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter) without success.

First attempt:

```rust
.filter(|x| **x > 0); 
error[E0614]: type `<I as std::iter::Iterator>::Item` cannot be dereferenced
  --> src/main.rs:13:21
   |
13 |         .filter(|x| **x > 0);
   |  
```

Second attempt:

```rust
.filter(|&x| *x > 0);
error[E0614]: type `<I as std::iter::Iterator>::Item` cannot be dereferenced
  --> src/main.rs:13:22
   |
13 |         .filter(|&x| *x > 0);
   |    
```

Why is the program not compiling?

answer1

The biggest issue (the other problem was covered by Lukas) is that you are comparing the iterator's generic items with an integer while comparisons provided by `PartialOrd`/`Ord` only work between the same types:

```rust
pub trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;
    ...
}
```

In order for a type `T` to be comparable with a number (in this case `0`), `T` must be a number and `0` needs to be of type `T` too. The `num` crate containing useful numeric traits can help here with its [`Zero`](https://docs.rs/num/0.2.0/num/trait.Zero.html) trait that provides a generic `0`:

```rust
extern crate num;

use num::Zero;

fn inspection<'a, I, T: 'a>(iter: I)
where
    I: Iterator<Item = &'a T>,
    T: Zero + PartialOrd // T is an ordered number
{
    let inspection = iter.filter(|&x| *x > T::zero()); // T::zero() is 0 of the same type as T
}

fn main() {
    let data = vec![1, 2, 3];
    inspection(data.iter());
}
```

You were really close to solving the first problem! You tried:

- `filter(|x| x > 0)`
- `filter(|x| **x > 0)`
- `filter(|&x| *x > 0)`

**The solution is: `filter(|x| \*x > 0)` \*or\* `filter(|&x| x > 0)`.**

To see why, let's take a close look at the error message again:

```none
9 |         .filter(|x| x > 0);
  |                         ^ expected reference, found integral variable
  |
  = note: expected type `&<I as std::iter::Iterator>::Item`
             found type `{integer}`
```

It says that it expected the type `&<I as std::iter::Iterator>::Item` and got the type `{integer}`. This means that the difference is *one* reference. You cannot compare `&i32` to `i32`. So you can remove the one `&` by using either `*` (the dereference operator) or the `|&x|` pattern (which dereferences the value too). But if you combine both, this has the same effect as `**`: it tries to dereferences twice. Which is not possible because the type only contains one reference.

------

However, it still doesn't compile due to a more complicated issue. You can read more about it here:

- [Trait for numeric functionality in Rust](https://stackoverflow.com/questions/37296351/trait-for-numeric-functionality-in-rust)
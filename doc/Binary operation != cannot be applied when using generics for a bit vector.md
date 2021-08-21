Binary operation != cannot be applied when using generics for a bit vector

I'm in the process of implementing a Bit Vector class as an exercise, however only knowing Rust for less than a week I run into trouble with the following code:

```rust
use std::cmp::Eq;
use std::ops::BitAnd;
use std::ops::Index;
use std::ops::Not;

struct BitVector<S = usize> 
    where S: Sized + BitAnd<usize> + Not + Eq {
    data: Vec<S>,
    capacity: usize
}

impl<S> BitVector<S>
    where S: Sized + BitAnd<usize> + Not + Eq {
    fn with_capacity(capacity: usize) -> BitVector {
        let len = (capacity / (std::mem::size_of::<S>() * 8)) + 1;
        BitVector { data: vec![0; len], capacity: capacity }
    }
}

impl<S> Index<usize> for BitVector<S>
    where S: Sized + BitAnd<usize> + Not + Eq {
    type Output = bool;

    fn index(&self, index: usize) -> &bool {
        let data_index = index / (std::mem::size_of::<S>() * 8);
        let remainder = index % (std::mem::size_of::<S>() * 8);
        (self.data[data_index] & (1 << remainder)) != 0
    }
}
```

The idea is that `S` can be one of for example `u8`, `u16`, `u32`, `u64` and `usize` to ensure that setting it to `0` in `with_capacity` creates a bit value for `S` that consists of all zeroes.

The error I get is the following:

> lib.rs:27:10: 27:50 error: binary operation `!=` cannot be applied to type `<S as std::ops::BitAnd<usize>>::Output` [E0369]
> lib.rs:27 (self.data[data_index] & (1 << remainder)) != 0
> ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
> lib.rs:27:10: 27:50 help: run `rustc --explain E0369` to see a detailed explanation
> lib.rs:27:10: 27:50 note: an implementation of `std::cmp::PartialEq` might be missing for `<S as std::ops::BitAnd<usize>>::Output`
> lib.rs:27 (self.data[data_index] & (1 << remainder)) != 0 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
> error: aborting due to previous error
> error: Could not compile `bit-vector`.

answer

This particular error here, in simple terms, means that the `Output` of `BitAnd`ing `S` and `usize` does not implement `PartialEq`. One fix would be to add a constraint that `S`'s `BitAnd<usize>`s `Output` is `S`:

```rust
BitAnd<usize, Output = S>
```

After this, you'll run into another error because you're comparing the value of the BitAnd to `0` and not to a value of type `S`. To fix that you can define your own `Zero` trait and use that or use Rust's unstable `std::num::Zero` and compare to `S::zero()`.

You'll also have to make `S: Copy` so that doing the `BitAnd` doesn't consume the the value (or add `S: Clone` and explicitly clone before calling `BitAnd::bitand`).

Finally you'll run into an error that your `index` must return a `&bool` while you're returning a `bool`. You can use the trick [bit-vec](https://github.com/contain-rs/bit-vec/blob/598b442a7769d0f4e8b19b53922ef7ec67318186/src/lib.rs#L169-L170) uses to define 2 statics:

```rust
static TRUE: bool = true;
static FALSE: bool = false;
```

and return `&TRUE` or `&FALSE` from `index`.

Final working (on Nightly) code:

```rust
#![feature(zero_one)]

use std::cmp::Eq;
use std::num::Zero;
use std::ops::BitAnd;
use std::ops::Index;
use std::ops::Not;

struct BitVector<S = usize>
    where S: Sized + BitAnd<usize, Output = S> + Not + Eq + Copy + Zero
{
    data: Vec<S>,
    capacity: usize,
}

impl<S> BitVector<S>
    where S: Sized + BitAnd<usize, Output = S> + Not + Eq + Copy + Zero
{
    fn with_capacity(capacity: usize) -> BitVector {
        let len = (capacity / (std::mem::size_of::<S>() * 8)) + 1;
        BitVector {
            data: vec![0; len],
            capacity: capacity,
        }
    }
}

static TRUE: bool = true;
static FALSE: bool = false;

impl<S> Index<usize> for BitVector<S>
    where S: Sized + BitAnd<usize, Output = S> + Not + Eq + Copy + Zero
{
    type Output = bool;

    fn index(&self, index: usize) -> &bool {
        let data_index = index / (std::mem::size_of::<S>() * 8);
        let remainder = index % (std::mem::size_of::<S>() * 8);
        if (self.data[data_index] & (1 << remainder)) != S::zero() {
            &TRUE
        } else {
            &FALSE
        }
    }
}

fn main() {
}
```


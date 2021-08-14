Default generic type parameter cannot be inferred

I'm trying to implement a Bit Vector library as an exercise, however I'm hitting trouble when wanting to define a default value for a generic type parameter.

This is an excerpt of the code I have:

```rust
extern crate num;

use std::cmp::Eq;
use std::ops::{BitAnd,BitOrAssign,Index,Shl};
use num::{One,Zero,Unsigned,NumCast};

pub trait BitStorage: Sized + 
    BitAnd<Self, Output = Self> + 
    BitOrAssign<Self> + 
    Shl<Self, Output = Self> + 
    Eq + Zero + One + Unsigned + NumCast + Copy {}

impl<S> BitStorage for S where S: Sized + 
    BitAnd<S, Output = S> + 
    BitOrAssign<S> + 
    Shl<S, Output = S> + 
    Eq + Zero + One + Unsigned + NumCast + Copy {}

pub struct BitVector<S: BitStorage = usize> {
    data: Vec<S>,
    capacity: usize
}

impl<S: BitStorage> BitVector<S> {
    pub fn with_capacity(capacity: usize) -> BitVector<S> {
        let len = (capacity / (std::mem::size_of::<S>() * 8)) + 1;
        BitVector { data: vec![S::zero(); len], capacity: capacity }
    }

    //...
}
```

And I want to use it as follows:

```rust
let vec = BitVector::with_capacity(1024);
```

However I get a compiler error:

> lib.rs:225:24: 225:48 error: unable to infer enough type information about `_`; type annotations or generic parameter binding required [E0282]
> lib.rs:225 let vec_1000 = BitVector::with_capacity(1000);
> ^~~~~~~~~~~~~~~~~~~~~~~~
> lib.rs:225:24: 225:48 help: run `rustc --explain E0282` to see a detailed explanation

To give a little more context to the code, currently valid types for `BitStorage` include (but are not limited to*) `u8`, `u16`, `u32`, `u64` and `usize`.

(*) I think you could write a custom `u128` implementation (just as example) if you implement all the traits for that type.

After googling about the issue I found [RFC 213](https://github.com/rust-lang/rfcs/blob/master/text/0213-defaulted-type-params.md) which does not seem to [be stable yet](https://github.com/rust-lang/rust/issues/27336). However on the other hand [HashMap](https://github.com/rust-lang/rust/blob/stable/src/libstd/collections/hash/map.rs) currently on stable is using default values, so it should be working, right?

answer1

The support for default type parameters is still limited, but can be used in some cases. When a `struct` with default type parameter is used to specify the type of a variable, the default type parameter is used to define the type of the variable:

```rust
// the type of vec is BitVector<usize>, so the type of
// BitVector::with_capacity(1024) is correctly inferred
let vec: BitVector = BitVector::with_capacity(1024);
```

> However on the other hand `HashMap` currently on stable is using default values, so it should be working, right?

Looking at the [`HashMap`](https://github.com/rust-lang/rust/blob/1.9.0/src/libstd/collections/hash/map.rs#L526-L539) source code, we can see that the methods `new` and `with_capacity` are implemented with `RandomState` for the `S` parameter and does not depend on the default type parameter in `HashMap`. All other methods are implemented as generic on `S`, including other "constructor" methods like `with_hasher`.

You can write something similar:

```rust
impl BitVector<usize> {
    pub fn default_with_capacity(capacity: usize) -> BitVector<usize> {
        // type is inferred
        Self::with_capacity(capacity)
    }
}

impl<S: BitStorage> BitVector<S> {
    pub fn with_capacity(capacity: usize) -> BitVector<S> {
        let len = (capacity / (std::mem::size_of::<S>() * 8)) + 1;
        BitVector {
            data: vec![S::zero(); len],
            capacity: capacity,
        }
    }

    // ...
}

// create with "default" BitStore
let vec = BitVector::default_with_capacity(1024);
// specify BitStore
let vec = BitVector::<u32>::with_capacity(1024);
```


How do I make a pointer hashable?

In Rust, I want to treat enums as equal, but still be able to distinguish different instances by pointer. Here's a toy example:

```rust
use self::Piece::*;
use std::collections::HashMap;

#[derive(Eq, PartialEq)]
enum Piece {
    Rook,
    Knight,
}

fn main() {
    let mut positions: HashMap<&Piece, (u8, u8)> = HashMap::new();
    let left_rook = Rook;
    let right_rook = Rook;

    positions.insert(&left_rook, (0, 0));
    positions.insert(&right_rook, (0, 7));
}
```

However, the compiler wants me to define `Hash` on `Piece`:

```none
error[E0277]: the trait bound `Piece: std::hash::Hash` is not satisfied
  --> src/main.rs:11:52
   |
11 |     let mut positions: HashMap<&Piece, (u8, u8)> = HashMap::new();
   |                                                    ^^^^^^^^^^^^ the trait `std::hash::Hash` is not implemented for `Piece`
   |
   = note: required because of the requirements on the impl of `std::hash::Hash` for `&Piece`
   = note: required by `<std::collections::HashMap<K, V>>::new`

error[E0599]: no method named `insert` found for type `std::collections::HashMap<&Piece, (u8, u8)>` in the current scope
  --> src/main.rs:15:15
   |
15 |     positions.insert(&left_rook, (0, 0));
   |               ^^^^^^
   |
   = note: the method `insert` exists but the following trait bounds were not satisfied:
           `&Piece : std::hash::Hash`

error[E0599]: no method named `insert` found for type `std::collections::HashMap<&Piece, (u8, u8)>` in the current scope
  --> src/main.rs:16:15
   |
16 |     positions.insert(&right_rook, (0, 7));
   |               ^^^^^^
   |
   = note: the method `insert` exists but the following trait bounds were not satisfied:
           `&Piece : std::hash::Hash`
```

I want equality defined on my enums so that one `Rook` is equal to another. However, I want to be able to distinguish different `Rook` instances in my `positions` hashmap.

How do I do this? I don't want to define `Hash` on `Piece`, but surely hashing is already defined on pointers?

answer

There's a difference between *raw pointers* (`*const T`, `*mut T`) and *references* (`&T`, `&mut T`) in Rust. You have a reference.

`Hash` [is defined](https://github.com/rust-lang/rust/blob/1.33.0/src/libcore/hash/mod.rs#L661-L673) for references as delegating to the hash of the referred-to item:

```rust
impl<T: ?Sized + Hash> Hash for &T {
    fn hash<H: Hasher>(&self, state: &mut H) {
        (**self).hash(state);
    }
}
```

However, it is [defined for raw pointers](https://github.com/rust-lang/rust/blob/1.33.0/src/libcore/hash/mod.rs#L675-L707) as you desire:

```rust
impl<T: ?Sized> Hash for *const T {
    fn hash<H: Hasher>(&self, state: &mut H) {
        if mem::size_of::<Self>() == mem::size_of::<usize>() {
            // Thin pointer
            state.write_usize(*self as *const () as usize);
        } else {
            // Fat pointer
            let (a, b) = unsafe {
                *(self as *const Self as *const (usize, usize))
            };
            state.write_usize(a);
            state.write_usize(b);
        }
    }
}
```

And that works:

```rust
let mut positions = HashMap::new();
positions.insert(&left_rook as *const Piece, (0, 0));
positions.insert(&right_rook as *const Piece, (0, 7));
```

However, using either references or raw pointers here is iffy at best.

If you use a reference, the compiler will stop you from using the hashmap once you have moved the values you have inserted as the reference will no longer be valid.

If you use raw pointers, the compiler *won't* stop you, but then you will have dangling pointers which can lead to memory unsafety.

In your case, I think I'd try to restructure the code so that a piece is unique beyond the memory address. Perhaps just some incremented number:

```rust
positions.insert((left_rook, 0), (0, 0));
positions.insert((right_rook, 1), (0, 7));
```

If that seems impossible, you can always `Box` the piece to give it a stable memory address. This latter solution is more akin to languages like Java where everything is heap-allocated by default.

------

As [Francis Gagné says](https://stackoverflow.com/questions/33847537/how-do-i-make-a-pointer-hashable/33847816#comment55460849_33847816):

> I'd rather wrap a `&'a T` in another struct that has the same identity semantics as `*const T` than have the lifetime erased

You can create a struct that handles *reference equality*:

```rust
#[derive(Debug, Eq)]
struct RefEquality<'a, T>(&'a T);

impl<'a, T> std::hash::Hash for RefEquality<'a, T> {
    fn hash<H>(&self, state: &mut H)
    where
        H: std::hash::Hasher,
    {
        (self.0 as *const T).hash(state)
    }
}

impl<'a, 'b, T> PartialEq<RefEquality<'b, T>> for RefEquality<'a, T> {
    fn eq(&self, other: &RefEquality<'b, T>) -> bool {
        self.0 as *const T == other.0 as *const T
    }
}
```

And then use that:

```rust
positions.insert(RefEquality(&left_rook), (0, 0));
positions.insert(RefEquality(&right_rook), (0, 7));
```
Why can't None be cloned for a generic Option when T doesn't implement Clone?

Given a struct with a generic `Option<T>` where `T` might not implement `Clone` why can't `None` be cloned? Isn't a `None` of type `T` the same as any other `None`? For example:

```rust
struct Foo<T> {
    bar: Vec<Option<T>>,
}

impl <T> Foo<T> {
    fn blank(size: usize) -> Foo<T> {
        Foo {
            bar: vec![None; size],
        }
    }
}
```

answer1

> Isn't a `None` of type `T` the same as any other `None`?

Definitely not! Unlike reference-based languages where null is typically implemented as a null-reference, Rust's `Option<T>` introduces no indirection and stores `T` inline when the option is `Some`. Since all enum variants have the same size, the `None` variant must still occupy at least as much space as `T`.

Having said that, it is technically true that the `None` value could be cloned without `T` being `Clone` simply because the `None` variant of the enum *doesn't* contain the `T`, it only stores the discriminator and reserves space that *could* contain `T` if the variant were to change to `Some`. But since Rust enum variants are not separate types, a trait bound defined for the enum must cover all variants.

See other answers more detailed explanations and instructions how to create a vector of `None` values of a non-cloneable `Option`.

answer2

As the other answers correctly point ouf, this is due to the way the `vec!`-macro is implemented. You can manually create a `Vec` of any `Option<T>` without requiring `T` to be `Clone`:

```rust
let bar = std::iter::repeat_with(|| Option::<T>::None).take(size).collect::<Vec<_>>();
```

This will create `size`-number of `Option::<T>::None` and place them in a `Vec`, which will be pre-allocated to the appropriate size. This works for any `T`.

Notes:

You can just use `repeat_with(|| None)`, it being an `Option<T>` is inferred.

answer3

An `Option` can only be cloned if the inner `T` implements `Clone`:

```rust
impl<T> Clone for Option<T>
where
    T: Clone, 
```

> Isn't a `None` of type `T` the same as any other `None`?

Actually, no. The Rust compiler does not even view `None` as a type. Instead, `None` is just a variant (subtype) of `Option<T>`. Try comparing `None`s of two different `T`s:

```rust
let a: Option<String> = None;
let b: Option<u8> = None;

assert_eq!(a, b)
```

You will see that they are in fact completely unrelated types:

```rust
error[E0308]: mismatched types
 --> src/main.rs:4:5
  |
4 |     assert_eq!(a, b)
  |     ^^^^^^^^^^^^^^^^ expected struct `String`, found `u8`
  |
  = note: expected enum `Option<String>`
             found enum `Option<u8>`
```

So the Rust compiler actually sees `None` as `Option::<T>::None`. This means that if `T` is not `Clone`, then `Option<T>` is not `Clone`, and therefore `Option::<T>::None` cannot be `Clone`.

To make your code compile, you must constrain `T` to `Clone`:

```rust
struct Foo<T: Clone> {
    bar: Vec<Option<T>>,
}

impl <T: Clone> Foo<T> {
    fn blank(size: usize) -> Foo<T> {
        Foo {
            bar: vec![None; size],
        }
    }
}
```

Now the compiler knows that `T` is `Clone`, and the implementation of `Clone` for `Option<T>` (and `Option::<T>::None`) is fulfilled.


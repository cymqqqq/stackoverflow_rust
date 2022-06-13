How to implement idiomatic operator overloading for values and references in Rust?

When implementing a primitive fixed-size vector type (`float2` for example), I want to support the `Add` and `Sub` traits. Later, I will want to support `Mul` and `*Assign`.

Looking up the documentation and other examples, I came up with this:

```rust
use std::ops::{Add, Sub};

#[derive(Copy, Clone)]
struct float2(f64, f64);

impl Add for float2 {
    type Output = float2;
    fn add(self, _rhs: float2) -> float2 {
        float2(self.0 + _rhs.0, self.1 + _rhs.1)
    }
}

impl Sub for float2 {
    type Output = float2;
    fn sub(self, _rhs: float2) -> float2 {
        float2(self.0 - _rhs.0, self.1 - _rhs.1)
    }
}
```

This works for basic examples, however I found in practice I would often end up with references passed in as arguments as well as local `float2`'s on the stack.

To mix these I needed to either:

- De-reference variables (OK but makes code a little less readable).
- Declare operator overloading combinations of references too.

Example:

```rust
impl<'a, 'b> Add<&'b float2> for &'a float2 {
    type Output = float2;
    fn add(self, _rhs: &'b float2) -> float2 {
        float2(self.0 + _rhs.0, self.1 + _rhs.1)
    }
}
impl<'a> Add<float2> for &'a float2 {
    type Output = float2;
    fn add(self, _rhs: float2) -> float2 {
        float2(self.0 + _rhs.0, self.1 + _rhs.1)
    }
}
impl<'b> Add<&'b float2> for float2 {
    type Output = float2;
    fn add(self, _rhs: &'b float2) -> float2 {
        float2(self.0 + _rhs.0, self.1 + _rhs.1)
    }
}

/*... and again for Sub */
```

While this allows to write expressions without de-referencing. it becomes quite tedious to enumerate each combinations, especially when adding more operations & types (`float3`, `float4`...).

Is there a generally accepted way to...

- Automatically coerce types for operator overloading?
- Use macros or some other feature of the language to avoid tedious repetition?

Or is it expected that developers either:

- Explicitly access variables as references as needed.
- Explicitly de-reference variables as needed.
- Write a lot of repetitive operator overloading functions.

------

*Note, I'm currently a beginner, I've checked some quite advanced math libraries in Rust, they're way over my head, while I could use them - I would like to understand how to write operator overloading for my own types.*

answer1

The great thing about Rust is that it's open source. This means you can see how the authors of the language have solved a problem. The closest analogue is [primitive integer types](https://github.com/rust-lang/rust/blob/1.10.0/src/libcore/ops.rs#L204-L216):

```rust
macro_rules! add_impl {
    ($($t:ty)*) => ($(
        #[stable(feature = "rust1", since = "1.0.0")]
        impl Add for $t {
            type Output = $t;

            #[inline]
            fn add(self, other: $t) -> $t { self + other }
        }

        forward_ref_binop! { impl Add, add for $t, $t }
    )*)
}
```

[`forward_ref_binop` is defined as](https://github.com/rust-lang/rust/blob/1.10.0/src/libcore/ops.rs#L133-L165):

```rust
macro_rules! forward_ref_binop {
    (impl $imp:ident, $method:ident for $t:ty, $u:ty) => {
        #[stable(feature = "rust1", since = "1.0.0")]
        impl<'a> $imp<$u> for &'a $t {
            type Output = <$t as $imp<$u>>::Output;

            #[inline]
            fn $method(self, other: $u) -> <$t as $imp<$u>>::Output {
                $imp::$method(*self, other)
            }
        }

        #[stable(feature = "rust1", since = "1.0.0")]
        impl<'a> $imp<&'a $u> for $t {
            type Output = <$t as $imp<$u>>::Output;

            #[inline]
            fn $method(self, other: &'a $u) -> <$t as $imp<$u>>::Output {
                $imp::$method(self, *other)
            }
        }

        #[stable(feature = "rust1", since = "1.0.0")]
        impl<'a, 'b> $imp<&'a $u> for &'b $t {
            type Output = <$t as $imp<$u>>::Output;

            #[inline]
            fn $method(self, other: &'a $u) -> <$t as $imp<$u>>::Output {
                $imp::$method(*self, *other)
            }
        }
    }
}
```

It's certainly valid to write wrapper implementations of the traits for references that simply dereference and call the value-oriented version.
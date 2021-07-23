Create method on iterator that returns iterator in Rust

I want to define a lazy `square()` method without unnecessary runtime overhead (no `dyn` keyword) that can be called on any `Iterable<Item = u8>` and returns another `Iterable<Item = u8>`, like so:

```rust
fn main() {
    vec![1, 2, 3, 4, 5]
        .iter()
        .filter(|x| x > 1)
        .squared()
        .filter(|x| x < 20);
}
```

I know how to define `squared()` as a standalone function:

```rust
fn squared<I: Iterator<Item = u8>>(iter: I) -> impl Iterator<Item = u8> {
    iter.map(|x| x * x)
}
```

To define that method on `Iterator<Item = u8>` though, I have to first define a `trait`. Here's where I struggle — traits cannot use the `impl` keyword in return values.

I'm looking for something like the following, which does not work:

```rust
trait Squarable<I: Iterator<Item = u8>> {
    fn squared(self) -> I;
}

impl<I, J> Squarable<I> for J
where
    I: Iterator<Item = u8>,
    J: Iterator<Item = u8>,
{
    fn squared(self) -> I {
        self.map(|x| x * x)
    }
}
```

I had many failed attempts at solving the problem, including changing the return type of `squared` to `Map<u8, fn(u8) -> u8>` and tinkering with `IntoIterable`s, but nothing worked so far. Any help would be greatly appreciated!

answer2

First of all, your output iterator should probably be an associated type and not a trait parameter, since that type is an *output* of the trait (it's not something that the caller can control).

```rust
trait Squarable {
    type Output: Iterator<Item = u8>;
    fn squared(self) -> I;
}
```

------

That being said, there are a few different possible approaches to solve this problem, each with different advantages and disadvantages.

## Using trait objects

The first is to use trait objects, e.g. `dyn Iterator<Item = u8>`, to erase the type at runtime. This comes at a slight runtime cost, but is definitely the simplest solution in stable Rust today:

```rust
trait Squarable {
    fn squared(self) -> Box<dyn Iterator<Item = u8>>;
}

impl<I: 'static + Iterator<Item = u8>> Squarable for I {
    fn squared(self) -> Box<dyn Iterator<Item = u8>> {
        Box::new(self.map(|x| x * x))
    }
}
```

## Using a custom iterator type

In stable rust, this is definitely the cleanest from the point of view of the user of the trait, however it takes a bit more code to implement because you need to write your own iterator type. However, for a simple `map` iterator this is pretty straight forward:

```rust
trait Squarable: Sized {
    fn squared(self) -> SquaredIter<Self>;
}

impl<I: Iterator<Item = u8>> Squarable for I {
    fn squared(self) -> SquaredIter<I> {
        SquaredIter(self)
    }
}

struct SquaredIter<I>(I);

impl<I: Iterator<Item = u8>> Iterator for SquaredIter<I> {
    type Item = u8;
    fn next(&mut self) -> Option<u8> {
        self.0.next().map(|x| x * x)
    }
}
```

## Using the explicit `Map` type

`<I as Iterator>::map(f)` has a type `std::iter::Map<I, F>`, so if the type `F` of the mapping function is known, we can use that type explicitly, at no runtime cost. This does expose the specific type as part of the function's return type though, which makes it harder to replace in the future without breaking dependent code. In most cases the function will also not be known; in this case we can use `F = fn(u8) -> u8` however since the function does not keep any internal state (but often that won't work).

```rust
trait Squarable: Sized {
    fn squared(self) -> std::iter::Map<Self, fn(u8) -> u8>;
}

impl<I: Iterator<Item = u8>> Squarable for I {
    fn squared(self) -> std::iter::Map<Self, fn(u8) -> u8> {
        self.map(|x| x * x)
    }
}
```

## Using an associated type

An alterative to the above is to give the trait an assoicated type. This still has the restriction that the function type must be known, but it's a bit more general since the `Map<...>` type is tied to the implementation instead of the trait itself.

```rust
trait Squarable {
    type Output: Iterator<Item = u8>;
    fn squared(self) -> Self::Output;
}

impl<I: Iterator<Item = u8>> Squarable for I {
    type Output = std::iter::Map<Self, fn(u8) -> u8>;
    fn squared(self) -> Self::Output {
        self.map(|x| x * x)
    }
}
```

## Using `impl` in associated type

This is similar to the "Using an associated type" above, but you can hide the actual type entirely, apart from the fact that it is an iterator. I personally think this is the preferrable solution, but unfortunately it is still unstable (it depends on the `type_alias_impl_trait` feature) so you can only use it in nightly Rust.

```rust
#![feature(type_alias_impl_trait)]

trait Squarable {
    type Output: Iterator<Item = u8>;
    fn squared(self) -> Self::Output;
}

impl<I: Iterator<Item = u8>> Squarable for I {
    type Output = impl Iterator<Item = u8>;
    fn squared(self) -> Self::Output {
        self.map(|x| x * x)
    }
}
```


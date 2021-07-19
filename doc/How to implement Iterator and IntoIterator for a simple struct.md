How to implement Iterator and IntoIterator for a simple struct?

How would someone implement the `Iterator` and `IntoIterator` traits for the following struct?

```rust
struct Pixel {
    r: i8,
    g: i8,
    b: i8,
}
```

I've tried various forms of the following with no success.

```rust
impl IntoIterator for Pixel {
    type Item = i8;
    type IntoIter = Iterator<Item=Self::Item>;

    fn into_iter(self) -> Self::IntoIter {
        [&self.r, &self.b, &self.g].into_iter()
    }
}
```

This code gives me a compile error

```none
error[E0277]: the trait bound `std::iter::Iterator<Item=i8> + 'static: std::marker::Sized` is not satisfied
 --> src/main.rs:7:6
  |
7 | impl IntoIterator for Pixel {
  |      ^^^^^^^^^^^^ the trait `std::marker::Sized` is not implemented for `std::iter::Iterator<Item=i8> + 'static`
  |
  = note: `std::iter::Iterator<Item=i8> + 'static` does not have a constant size known at compile-time
  = note: required by `std::iter::IntoIterator`
```

answer1

Your iterator type is `Iterator<Item = Self::Item>`, but `Iterator` is a trait. Traits are *implemented* by structs, they don't exist on their own. You could also have a reference trait object (`&Iterator`), a boxed trait object (`Box<Iterator>`) or an anonymous trait implementation (`impl Iterator`), all of which have a known sizes.

Instead, we create a `PixelIntoIterator` that has a known size and *implements* `Iterator` itself:

```rust
struct Pixel {
    r: i8,
    g: i8,
    b: i8,
}

impl IntoIterator for Pixel {
    type Item = i8;
    type IntoIter = PixelIntoIterator;

    fn into_iter(self) -> Self::IntoIter {
        PixelIntoIterator {
            pixel: self,
            index: 0,
        }
    }
}

pub struct PixelIntoIterator {
    pixel: Pixel,
    index: usize,
}

impl Iterator for PixelIntoIterator {
    type Item = i8;
    fn next(&mut self) -> Option<i8> {
        let result = match self.index {
            0 => self.pixel.r,
            1 => self.pixel.g,
            2 => self.pixel.b,
            _ => return None,
        };
        self.index += 1;
        Some(result)
    }
}

fn main() {
    let p = Pixel {
        r: 54,
        g: 23,
        b: 74,
    };
    for component in p {
        println!("{}", component);
    }
}
```

This has the nice benefit of returning actual `i8`s, not references. Since these are so small, you might as well pass them directly.

This consumes the `Pixel`. If you had a reference to a `Pixel`, you'd need to also implement an iterator that doesn't consume it:

```rust
impl<'a> IntoIterator for &'a Pixel {
    type Item = i8;
    type IntoIter = PixelIterator<'a>;

    fn into_iter(self) -> Self::IntoIter {
        PixelIterator {
            pixel: self,
            index: 0,
        }
    }
}

pub struct PixelIterator<'a> {
    pixel: &'a Pixel,
    index: usize,
}

impl<'a> Iterator for PixelIterator<'a> {
    type Item = i8;
    fn next(&mut self) -> Option<i8> {
        let result = match self.index {
            0 => self.pixel.r,
            1 => self.pixel.g,
            2 => self.pixel.b,
            _ => return None,
        };
        self.index += 1;
        Some(result)
    }
}
```

If you wanted to support creating both a consuming iterator and a non-consuming iterator, you can implement both versions. You can always take a reference to a `Pixel` you own, so you only *need* the non-consuming variant. However, it's often nice to have a consuming version so that you can return the iterator without worrying about lifetimes.

------

> it'd be much more convenient to write this by reusing iterators that already exists, e.g., with `[T; 3]`

As of Rust 1.51, you can leverage [`array::IntoIter`](https://doc.rust-lang.org/std/array/struct.IntoIter.html):

```rust
impl IntoIterator for Pixel {
    type Item = i8;
    type IntoIter = std::array::IntoIter<i8, 3>;

    fn into_iter(self) -> Self::IntoIter {
        std::array::IntoIter::new([self.r, self.b, self.g])
    }
}
```

In previous versions, it might be a bit silly, but you could avoid creating your own iterator type by gluing some existing types together and using `impl Iterator`:

```rust
use std::iter;

impl Pixel {
    fn values(&self) -> impl Iterator<Item = i8> {
        let r = iter::once(self.r);
        let b = iter::once(self.b);
        let g = iter::once(self.g);
        r.chain(b).chain(g)
    }
}
```

answer2

First, `IntoIter` must point to a real `struct` and not to a `trait` in order for Rust to be able to pass the value around (that's what `Sized` means). In case of arrays `into_iter` returns the [std::slice::Iter](http://doc.rust-lang.org/std/slice/struct.Iter.html) `struct`.

Second, a typical array, `[1, 2, 3]`, isn't allocated on heap. In fact, the compiler is allowed to optimize away the allocation entirely, pointing to a pre-compiled array instead. Being able to iterate the arrays without copying them anywhere is I think the reason why the `IntoIterator` implementation for arrays doesn't *move* the array anywhere as other `IntoIterator` implementations do. Instead it seems to *reference* the existing array. You can see from [its signature](http://doc.rust-lang.org/std/primitive.array.html)

```rust
impl<'a, T> IntoIterator for &'a [T; 3]
    type Item = &'a T
    type IntoIter = Iter<'a, T>
    fn into_iter(self) -> Iter<'a, T>
```

that it takes a *reference* to an array (`&'a [T; 3]`).

As such, you can't use it in the way you're trying to. The referenced array must outlive the returned iterator. [Here's a version](http://is.gd/aXLu3K) where Rust compiler tells so.

Vector has an `IntoIterator` implementation that truly moves the data into the iterator and so [you can use it](http://is.gd/dktJAa).

------

P.S. To make it both fast and simple, return an array instead of an iterator ([playpen](http://is.gd/gWhlYX)):

```rust
impl Pixel {
    fn into_array(self) -> [i8; 3] {[self.r, self.g, self.b]}
}
```

That way the array is first *moved* into the outer scope and then it can be *referenced* from the outer scope's iterator:

```rust
for color in &(Pixel {r: 1, g: 2, b: 3}).into_array() {
    println! ("{}", color);
}
```


How to call `min` on a generic type that could be either an integer or float?

What do I do when I want to call min on integers and floats? For example consider this:

```rust
fn foo<T>(v1: T, v2: T)
    where ???
{
   ....
   let new_min = min(v1, v2);
   ....
}
```

The problem is that [min](https://doc.rust-lang.org/std/cmp/fn.min.html) doesn't work for `f32`. There is [another min](http://rust-num.github.io/num/num/trait.Float.html#tymethod.min) for floats.

How would I solve this problem?

answer1

Create your own trait that defines the behavior of the various types:

```rust
trait Min {
    fn min(self, other: Self) -> Self;
}

impl Min for u8 {
    fn min(self, other: u8) -> u8 { ::std::cmp::min(self, other) }
}

impl Min for f32 {
    fn min(self, other: f32) -> f32 { f32::min(self, other) }
}

fn foo<T>(v1: T, v2: T)
    where T: Min
{
   let new_min = Min::min(v1, v2);
}
```

As [mentioned](https://stackoverflow.com/questions/28247990/how-to-do-a-binary-search-on-a-vec-of-floats) in [other places](https://stackoverflow.com/questions/26489701/in-rust-f64-and-f32-dont-implement-total-ordering-via-ord-trait-why-this-restr), floating point comparisons [are hard](http://floating-point-gui.de/).

There's no **one** answer to what the result of `min(NaN, 0.0)` should be, so it's up to you to decide. If you decide that `NaN` is less than or greater than all other numbers, great! Maybe it's equal to zero! Maybe you should assert that there will *never* be a `NaN`...


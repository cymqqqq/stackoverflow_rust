In Rust, is a vector an Iterator?

Is it accurate to state that a vector (among other collection types) is an `Iterator`?

For example, I can loop over a vector in the following way, because it implements the `Iterator` trait (as I understand it):

```rust
let v = vec![1, 2, 3, 4, 5];

for x in &v {
    println!("{}", x);
}
```

However, if I want to use functions that are part of the `Iterator` trait (such as `fold`, `map` or `filter`) why must I first call `iter()` on that vector?

Another thought I had was maybe that a vector can be converted into an `Iterator`, and, in that case, the syntax above makes more sense.

answer1

No, a vector is not an iterator.

But it implements the trait [`IntoIterator`](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html), which the `for` loop uses to convert the vector into the required iterator.

In the [documentation for `Vec`](https://doc.rust-lang.org/std/vec/struct.Vec.html) you can see that `IntoIterator` is implemented in three ways:

- for `Vec<T>`, which is moved and the iterator returns items of type `T`,
- for a shared reference `&Vec<T>`, where the iterator returns shared references `&T`,
- and for `&mut Vec<T>`, where mutable references are returned.

[`iter()`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter) is just a method in `Vec` to convert `Vec<T>` directly into an iterator that returns shared references, without first converting it into a reference. There is a sibling method [`iter_mut()`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter_mut) for producing mutable references.


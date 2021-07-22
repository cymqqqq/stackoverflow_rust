Why doesn't Vec implement the Iterator trait?

What is the design reason for `Vec` not implementing the `Iterator` trait? Having to always call `iter()` on all vectors and slices makes for longer lines of code.

Example:

```rust
let rx = xs.iter().zip(ys.iter());
```

compared to Scala:

```scala
val rx = xs.zip(ys)
```

answer1

An iterator has an iteration state. It must know what will be the next element to give you.

So a vector by itself isn't an iterator, and the distinction is important. You can have two iterators over the same vector, for example, each with its specific iteration state.

But a vector can provide you an iterator, that's why it implements [`IntoIterator`](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html), which lets you write this:

```rust
let v = vec![1, 4];
for a in v {
    dbg!(a);
}
```

Many functions take an `IntoIterator` when an iterator is needed, and that's the case for `zip`, which is why

```rust
let rx = xs.iter().zip(ys.iter());
```

can be replaced with

```rust
let rx = xs.iter().zip(ys);
```

answer2

> What is the design reason for `Vec` not implementing the `Iterator` trait?

Which of the three iterators should it implement? There are three different kinds of iterator you can get from a `Vec`:

1. `vec.iter()` gives `Iterator<Item = &T>`,
2. `vec.iter_mut()` gives `Iterator<Item = &mut T>` and modifies the vector and
3. `vec.into_iter()` gives `Iterator<Item = T>` and consumes the vector in the process.

> compared to Scala:

In Scala it does not implement `Iterator` directly either, because `Iterator` needs the next item pointer that the vector itself does not have. However since Scala does not have move semantics, it only has one way to create an iterator from a vector, so it can do the conversion implicitly. Rust has three methods, so it must ask you which one you want.


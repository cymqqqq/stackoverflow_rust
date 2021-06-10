What is the difference between iter and into_iter?

- The iterator returned by `into_iter` may yield any of `T`, `&T` or `&mut T`, depending on the context.
- The iterator returned by `iter` will yield `&T`, by convention.
- The iterator returned by `iter_mut` will yield `&mut T`, by convention.

ANSWER1

The first question is: "What is `into_iter`?"

`into_iter` comes from the [`IntoIterator` trait](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html):

```rust
pub trait IntoIterator
where
	<Self::IntoIter as Iterator>::Item == Self::Item,
{
    type Item;
   	type IntoItem: Iterator;
    fn into_iter(self) -> Self::IntoIter;
}
```

You implement this trait when you want to specify how a particular type is to be converted into an iterator. Most notably, if a type implements `IntoIterator` it can be used in a `for` loop.

For example, `Vec` implements `IntoIterator`...

```rust
impl<T> IntoIterator for Vec<T>
impl<'a,T> IntoIterator for &'a Vec<T>
impl<'a, T> IntoIterator for &'a mut Vec<T>
```

Each variant is slightly different.

This one consumes the `Vec` and its iterator [yields **values**](https://doc.rust-lang.org/std/vec/struct.Vec.html#impl-IntoIterator) (`T` directly):

```rust
impl<T> IntoIterator for Vec<T> {
    type Item = T;
    type IntoIter = IntoIter<T>;
    fn into_iter(mut self) -> IntoIter<T> {}
}
```

The other two take the vector by reference (don't be fooled by the signature of `into_iter(self)` because `self` is a reference in both cases) and their iterators will produce references to the elements inside `Vec`.

This one [yields **immutable references**](https://doc.rust-lang.org/std/vec/struct.Vec.html#impl-IntoIterator-1):

```rust
impl<'a, T> IntoIterator for &'a Vec<T> {
    type Item = &'a T;
    type IntoIter = slice::Iter<'a, T>;
    fn into_iter(self) -> slice::Iter<'a, T> {}
}
```

While this one [yields **mutable references**](https://doc.rust-lang.org/std/vec/struct.Vec.html#impl-IntoIterator-2):

```rust
impl<'a, T> IntoIterator for &'a mut Vec<T> {
    type Item = &'a mut T;
    type IntoIter = slice::IterMut<'a, T>;
    fn into_iter(self) -> slice::IterMut<'a, T> {}
}
```

So:

> What is the difference between `iter` and `into_iter`?

`into_iter` is a generic method to obtain an iterator, whether this iterator yields values, immutable references or mutable references **is context dependent** and can sometimes be surprising.

`iter` and `iter_mut` are ad-hoc methods. Their return type is therefore independent of the context, and will conventionally be iterators yielding immutable references and mutable references, respectively.

ANSWER2

- `iter()` iterates over the items by reference
- `into_iter()` iterates over the items, moving them into the new scope
- `iter_mut()` iterates over the items, giving a mutable reference to each item

So `for x in my_vec { ... }` is essentially equivalent to `my_vec.into_iter().for_each(|x| ... )` - both `move` the elements of `my_vec` into the `...` scope.

If you just need to "look at" the data, use `iter`, if you need to edit/mutate it, use `iter_mut`, and if you need to give it a new owner, use `into_iter`.
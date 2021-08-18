How can I create a HashSet from a borrowed array of a generic type in Rust?

I have a function taking two borrowed arrays of a generic type `T`, and I would like to create `HashSet`s from those arrays so I can compare them.

I thought I would be able to do something like this:

```rust
pub fn sublist<T: PartialEq>(first_list: &[T], second_list: &[T]) -> bool {
  let first_set: HashSet<T> = first_list.iter().collect();
  let second_set: HashSet<T> = second_list.iter().collect();

  first_set.is_subset(&second_set)
}
```

But I end up with the following errors:

```rust
a value of type `std::collections::HashSet<T>` cannot be built from an iterator over elements of type `&T`

value of type `std::collections::HashSet<T>` cannot be built from `std::iter::Iterator<Item=&T>`

help: the trait `std::iter::FromIterator<&T>` is not implemented for `std::collections::HashSet<T>`
```

Because of the first line in the error, I thought I might be able to solve it like this (I just changed the hashset type to references `&T`):

```rust
pub fn sublist<T: PartialEq>(first_list: &[T], second_list: &[T]) -> bool {
  let first_set: HashSet<&T> = first_list.iter().collect();
  let second_set: HashSet<&T> = second_list.iter().collect();

  first_set.is_subset(&second_set)
}
```

But then I see these errors:

```rust
the trait bound `T: std::cmp::Eq` is not satisfied

the trait `std::cmp::Eq` is not implemented for `T`

note: required because of the requirements on the impl of `std::cmp::Eq` for `&T`
note: required because of the requirements on the impl of `std::iter::FromIterator<&T>` for `std::collections::HashSet<&T>`
```

I don't understand how to create new data structures from references to an array. Is it the fact that these arrays are borrowed that is the problem, or is it ultimately the trait bound on `PartialEq` that is the problem?

What if, for whatever reason, I can't modify the function signature, how can I use hashsets to compare the collections?

answer

To use `HashSet` your function needs to have the `Eq` and `Hash` trait bounds:

```rust
use std::hash::Hash;
use std::collections::HashSet;

pub fn sublist<T: Eq + Hash>(first_list: &[T], second_list: &[T]) -> bool {
  let first_set: HashSet<&T> = first_list.iter().collect();
  let second_set: HashSet<&T> = second_list.iter().collect();

  first_set.is_subset(&second_set)
}
```

If you only know that `T` is `PartialEq`, then you can implement it like so:

```rust
pub fn sublist<T: PartialEq>(first_list: &[T], second_list: &[T]) -> bool {
    first_list.iter().all(|v| second_list.contains(v))
}
```

Other options include `T: Ord` and use `BTreeSet`.


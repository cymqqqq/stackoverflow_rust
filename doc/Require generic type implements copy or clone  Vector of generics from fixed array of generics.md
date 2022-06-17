Require generic type implements copy or clone | Vector of generics from fixed array of generics

I am trying to implement a custom data structure in Rust that behaves like a Set in mathematics (supports Union, Intersection, Disjoint Comparison, etc.) I want the constructor (associated function new) to take a slice of generic type `T` and populate a Vector of type `T`.

```rust
#[derive(Debug, PartialEq)]
pub struct CustomSet<T> {
    elements: Vec<T>
}

impl<T> CustomSet<T> {
    pub fn new(input: &[T]) -> Self {
       let mut elements = Vec::new();
       for elm in input {
           elements.push(*elm);
       }
       CustomSet {
           elements
       }
    }
}
```

This makes the compiler very upset, because `T` does not necessarily implement `Copy` (or `Clone` for that matter). I can't manually implement `Copy` or `Clone` on `T` because I have no idea what `T` is. I'd prefer not to use a `Vec<&T>` because I don't really want to manage lifetimes. I think the solution is to somehow require that `T` implement `Clone` or `Copy`, which is fine for my purposes, but I also don't know how to do that. Any and all advice on this matter is welcome.

answer

`Copy` constraint in generic impl:

```rust
#[derive(Debug, PartialEq)]
pub struct CustomSet<T> {
    elements: Vec<T>
}

impl<T: Copy> CustomSet<T> {
    pub fn new(input: &[T]) -> Self {
       CustomSet {
           elements: input.to_vec()
       }
    }
}
```

`Clone` constraint in generic impl:

```rust
#[derive(Debug, PartialEq)]
pub struct CustomSet<T> {
    elements: Vec<T>
}

impl<T: Clone> CustomSet<T> {
    pub fn new(input: &[T]) -> Self {
       CustomSet {
           elements: input.to_vec()
       }
    }
}
```


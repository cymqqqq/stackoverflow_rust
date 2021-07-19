Does Rust have a way to apply a function/method to each element in an array or vector?

Does the Rust language have a way to apply a function to each element in an array or vector?

I know in Python there is the `map()` function which performs this task. In R there is the `lapply()`, `tapply()`, and `apply()` functions that also do this.

Is there an established way to vectorize a function in Rust?

notes

Here you go: [doc.rust-lang.org/std/iter/trait.Iterator.html#method.map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map) You can use it like: `vec![1, 2, 3].into_iter().map(|x| x * 2).collect::<Vec<_>>()`

answer1

Rust has [`Iterator::map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map), so you can:

```rust
some_vec.iter().map(|x| /* do something here */)
```

However, `Iterator`s are lazy so this won't do anything by itself. You can tack a [`.collect()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect) onto the end to make a new vector with the new elements, if that's what you want:

```rust
let some_vec = vec![1, 2, 3];
let doubled: Vec<_> = some_vec.iter().map(|x| x * 2).collect();
println!("{:?}", doubled);
```

The standard way to perform side effects is to use a `for` loop:

```rust
let some_vec = vec![1, 2, 3];
for i in &some_vec {
    println!("{}", i);
}
```

If the side effect should modify the values in place, you can use an iterator of mutable references:

```rust
let mut some_vec = vec![1, 2, 3];
for i in &mut some_vec {
    *i *= 2;
}
println!("{:?}", some_vec); // [2, 4, 6]
```

If you really want the functional style, you can use the [`.for_each()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.for_each) method:

```rust
let mut some_vec = vec![1, 2, 3];
some_vec.iter_mut().for_each(|i| *i *= 2);
println!("{:?}", some_vec); // [2, 4, 6]
```

answer2

Since Rust 1.21, the `std::iter::Iterator` trait defines a [`for_each()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.for_each) combinator which can be used to apply an operation to each element in the collection. It is eager (not lazy), so `collect()` is not needed:

```rust
fn main() {
    let mut vec = vec![1, 2, 3, 4, 5];
    vec.iter_mut().for_each(|el| *el *= 2);
    println!("{:?}", vec);
}
```

The above code prints `[2, 4, 6, 8, 10]` to the console.

[Rust playground](https://play.rust-lang.org/?gist=d7111b224a82862422aa712c3b7a0596&version=stable&mode=debug)


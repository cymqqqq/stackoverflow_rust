C++ example:

```rust
for (long i = 0; i < 101; i++) {
    //...
}
```

In Rust I tried:

```rust
for i: i64 in 1..100 {
    // ...
}
```

I could easily just declare a `let i: i64 =` var before the for loop but I'd rather learn the correct way to doing this, but this resulted in

```rustnone
error: expected one of `@` or `in`, found `:`
 --> src/main.rs:2:10
  |
2 |     for i: i64 in 1..100 {
  |          ^ expected one of `@` or `in` here
```

You can use an [integer suffix](https://doc.rust-lang.org/stable/reference/tokens.html#numbers) on one of the literals you've used in the range. Type inference will do the rest:

```rust
for i in 1i64..101 {
    println!("{}", i);
}
```

Suffixes for numeric literals are built into the language. 

it is **not** possible to declare the type of the variable in a `for` loop.

Instead, a more general approach (e.g. applicable also to `enumerate()`) is to introduce a `let` binding by destructuring the item inside the body of the loop.

Example:

```rust
for e in bytes.iter().enumerate() {
    let (i, &item): (usize, &u8) = e; // here
    if item == b' ' {
        return i;
    }
}
```

You can also use `(_, &u8)` if you want to omit the `usize` part 

If your loop variable happens to be the result of a function call that returns a generic type:

```rust
let input = ["1", "two", "3"];
for v in input.iter().map(|x| x.parse()) {
    println!("{:?}", v);
}
error[E0284]: type annotations required: cannot resolve `<_ as std::str::FromStr>::Err == _`
 --> src/main.rs:3:37
  |
3 |     for v in input.iter().map(|x| x.parse()) {
  |                                     ^^^^^
```

You can use a *turbofish* to specify the types:

```rust
for v in input.iter().map(|x| x.parse::<i32>()) {
//                                   ^^^^^^^
    println!("{:?}", v);
}
```

Or you can use the *fully-qualified syntax*:

```rust
for v in input.iter().map(|x| <i32 as std::str::FromStr>::from_str(x)) {
//                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    println!("{:?}", v);
}
```
Rust macro for repeating array elements

write a Rust macro that fills up an array with repeating elements, in this case with zeros. This is what I came up with:

```rust
macro_rules! pad4  {
    () => {
        println!("0b00000000, 0b00000000, 0b00000000, 0b00000000");
    }
}

const arr: [u8; 8] = [pad4!(), 0b01111100, 0b10000010, 0b00000010, 0b01111110];
```

But I'm getting the following error:

```none
expected `u8`, found `()`
```

Rust macros aren't simple string replacements, they pattern match over parsed tokens and must return Rust syntax that is valid in the context the macro is invoked in.

Your current macro:

```rust
macro_rules! pad4  {
    () => {
        println!("0b00000000, 0b00000000, 0b00000000, 0b00000000");
    }
}
```

Called in this context:

```rust
const arr: [u8; 8] = [pad4!(), 0b01111100, 0b10000010, 0b00000010, 0b01111110];
```

Expands to this:

```rust
const arr: [u8; 8] = [
    {
        println!("0b00000000, 0b00000000, 0b00000000, 0b00000000");
    },
    0b01111100,
    0b10000010,
    0b00000010,
    0b01111110,
];
```

Which is why you're getting an error, as the first expression block in the array returns `()` instead of the expected `u8`.

You can use e.g. [`cargo expand`](https://crates.io/crates/cargo-expand) to easily inspect the result of macro expansion.

------

Here's `pad4` but written in a way that works:

```rust
macro_rules! pad4 {
    [$($e:expr),*] => {
        [0b00000000, 0b00000000, 0b00000000, 0b00000000, $($e,)*]
    }
}

const arr: [u8; 8] = pad4![0b01111100, 0b10000010, 0b00000010, 0b01111110];
```
Pointers to rows in Rust

How can I get pointer to first row of two dimensional array in Rust? And how can I pass the pointer to function so values in the row can be changed?

This is how I would make an array:

```rust
let state = [mut [mut 0u8, ..4], ..4];
```

answer

This should do:

```rust
fn change_one_row(x: &[mut u8]) {
   x[0] = 5;
}

fn main() {
    let state = [mut [mut 0u8, ..4], ..4];
    change_one_row(state[2]);
    io::println(fmt!("%u", state[2][0] as uint))
}
```


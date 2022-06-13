How to operate on 2 mutable slices of a Rust array?

I have a function that needs to operate on two parts of a single array. The purpose is to be able to build an `#[nostd]` allocator that can return a variable slice of a bigger array to the caller and hang on to the remainder of the array for future allocations.

Here's example code that fails:

```rust
fn split<'a>(mut item: &'a mut [i32], place: usize) -> (&'a mut [i32], &'a mut [i32]) {
    (&mut item[0..place], &mut item[place..])
}

fn main() {
    let mut mem: [i32; 2048] = [1; 2048];
    let (mut array0, mut array1) = split(&mut mem[..], 768);
    array0[0] = 4;
    println!("{:?} {:?}", array0[0], array1[0]);
}
```

the error is as follows:

```none
error[E0499]: cannot borrow `*item` as mutable more than once at a time
 --> src/main.rs:2:32
  |
2 |     (&mut item[0..place], &mut item[place..])
  |           ----                 ^^^^ second mutable borrow occurs here
  |           |
  |           first mutable borrow occurs here
3 | }
  | - first borrow ends here
```

This pattern also can be helpful for in-place quicksort, etc.

Is there anything unsafe about having two mutable references to nonoverlapping slices of the same array? If there's no way in pure Rust, is there a "safe" `unsafe` incantation that will allow it to proceed?

Is there anything unsafe about having two mutable references to nonoverlapping slices of the same array?

There isn't, but Rust's type system cannot currently detect that you're taking mutable references to two non-overlapping parts of a slice. As this is a common use case, Rust provides a safe function to do exactly what you want: [`std::slice::split_at_mut`](https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut).

> ```
> fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T])
> ```
>
> Divides one `&mut` into two at an index.
>
> The first will contain all indices from `[0, mid)` (excluding the index `mid` itself) and the second will contain all indices from `[mid, len)` (excluding the index `len` itself).

The final code is:

```rust
fn main() {
    let mut mem : [i32; 2048] = [1; 2048];
    let (mut array0, mut array1) = mem[..].split_at_mut(768);
    array0[0] = 4;
    println!("{:?} {:?}", array0[0], array1[0]);
}
```
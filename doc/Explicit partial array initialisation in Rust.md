Explicit partial array initialisation in Rust

In C, I can write `int foo[100] = { 7, 8 };` and I will get `[7, 8, 0, 0, 0...]`.

This allows me to explicitly and concisely choose initial values for a contiguous group of elements at the beginning of the array, and the remainder will be initialised as if they had static storage duration (i.e. to the zero value of the appropriate type).

Is there an equivalent in Rust?

 there is no such shortcut. You do have a few options, though.

------

**The direct syntax**

The direct syntax to initialize an array works with `Copy` types (integers are `Copy`):

```rust
let array = [0; 1024];
```

initializes an array of 1024 elements with all 0s.

Based on this, you can afterwards modify the array:

```rust
let array = {
    let mut array = [0; 1024];
    array[0] = 7;
    array[1] = 8;
    array
};
```

*Note the trick of using a block expression to isolate the mutability to a smaller section of the code; we'll reuse it below.*

------

**The iterator syntax**

There is also support to initialize an array from an iterator:

```rust
let array = {
    let mut array = [0; 1024];

    for (i, element) in array.iter_mut().enumerate().take(2) {
        *element = (i + 7);
    }

    array
};
```

And you can even (optionally) start from an uninitialized state, using an `unsafe` block:

```rust
let array = unsafe {
    // Create an uninitialized array.
    let mut array: [i32; 10] = mem::uninitialized();

    let nonzero = 2;

    for (i, element) in array.iter_mut().enumerate().take(nonzero) {
        // Overwrite `element` without running the destructor of the old value.
        ptr::write(element, i + 7)
    }

    for element in array.iter_mut().skip(nonzero) {
        // Overwrite `element` without running the destructor of the old value.
        ptr::write(element, 0)
    }

    array
};
```

------

**The shorter iterator syntax**

There is a shorter form, based on [`clone_from_slice`](http://doc.rust-lang.org/std/primitive.slice.html#method.clone_from_slice), it is currently unstable however.

```rust
#![feature(clone_from_slice)]

let array = {
    let mut array = [0; 32];

    // Override beginning of array
    array.clone_from_slice(&[7, 8]);

    array
};
```

macro

```rust
macro_rules! array {
    ($($v:expr),*) => (
        {
            let mut array = Default::default();
            {
                let mut e = <_ as ::std::convert::AsMut<[_]>>::as_mut(&mut array).iter_mut();
                $(*e.next().unwrap() = $v);*;
            }
            array
        }
    )
}

fn main() {
    let a: [usize; 5] = array!(7, 8);
    assert_eq!([7, 8, 0, 0, 0], a);
}
```
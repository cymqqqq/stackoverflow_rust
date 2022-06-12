Calling Rust method from C with array parameters

I'm trying to call Rust code from my C project for an embedded device. The device prints over UART, so I am able to see what the result of my call is.

The following C and Rust code works as expected (I have omitted a lot of boilerplate Rust code that is needed to make it compile).

C:

```C++
uint8_t input[] = {1,2,3};
uint8_t output[] = {4,5,6};
output = func(input, output);
printf("Sum: %d", output[0]);
```

Rust:

```Rust
#[no_mangle]
pub extern fn func(input: &[u8], dst: &mut[u8]) -> u8 {
  3
}
```

This prints 3 as expected. But I'm stuck at mutating the arrays passed in as references:

C:

```C++
uint8_t input[] = {1,2,3};
uint8_t output[] = {4,5,6};
func(input, output);
printf("Sum: %d", output[0]);
```

Rust:

```Rust
#[no_mangle]
pub extern fn func(input: &[u8], dst: &mut[u8]) {
  for i in (0..1) {
      dst[i] = input[i];
  }
}
```

This compiles, but prints 4 instead of the expected 1. For some reason I'm not able to change the value of the array. Any ideas?

EDIT: The C function declarations are respectively:

```
extern uint8_t func(uint8_t in[64], uint8_t output[64]);
extern void func(uint8_t in[64], uint8_t output[64]);
```

EDIT2: Updated code: C:

```C++
uint8_t input[64];
uint8_t output[64];
for(uint8_t = 0; i < 64; i++) {
    input[i] = i;
}
func(input, output);
printf("Sum: %d", output[2]);
```

Expects output 2.

answer1

A `&[T]` in Rust **is not the same thing** as a `T []` or a `T *` in C. You should **never** use borrowed pointers for interacting with C code from Rust. You should also **never, ever** use `[T]` or `str` when interacting with C code.

**Ever**.

`[T]` and `str` are [dynamically sized types](http://doc.rust-lang.org/book/unsized-types.html), meaning that all pointers to them (of any kind) are twice the size of a regular pointer. This means that your C code is passing two pointers, whereas Rust is expecting *four*. It's a small miracle your second example didn't just explode in your face.

The [Slice Arguments example from the Rust FFI Omnibus](http://jakegoulding.com/rust-ffi-omnibus/slice_arguments/) is very nearly exactly what you want.

There is also the [FFI chapter of the Rust Book](http://doc.rust-lang.org/book/ffi.html).

**Edit**: Those C signatures are also bogus; first of all, there is no limit on the size of the arrays Rust will accept anywhere, so I'm not sure where `64` came from. A vaguely comparable Rust type would be `[u8; 64]`, but even *that* would *still* be incorrect, because C and Rust pass fixed-size arrays *differently.* C passes them by-reference, Rust passes them by-value.

**Edit 2**: assuming you're talking about the second `func`, the Rust translation is just:

```rust
// C ffi signature:
// void copy(uint8_t src[4], uint8_t dst[4]);
#[no_mangle]
pub unsafe extern fn copy(src: *const [u8; 4], dst: *mut [u8; 4]) {
    if src.is_null() { return; }
    if dst.is_null() { return; }

    // Convert to borrowed pointers.
    let src: &[u8; 4] = &*src;
    let dst: &mut [u8; 4] = &mut *dst;

    for (s, d) in src.iter().zip(dst.iter_mut()) {
        *d = *s;
    }
}

#[cfg(test)]
#[test]
fn test_copy() {
    let a = [0, 1, 2, 3];
    let mut b = [0; 4];
    unsafe { copy(&a, &mut b); }
    assert_eq!(b, [0, 1, 2, 3]);
}
```

notes：

If C passes such arrays by reference, then the Rust signatures would need to use `*const [uint8_t; 64]` or `*mut [uint8_t; 64]`.

 A bigger problem is that they're *not* 64-element arrays in the first place, but yes; the Rust equivalent of `uint8_t [64]` as an argument is (at least on platforms where I've tested it) `*mut [u8; 64]`.
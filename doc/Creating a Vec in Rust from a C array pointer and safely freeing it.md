Creating a Vec in Rust from a C array pointer and safely freeing it?

I'm calling a C function from Rust which takes a null pointer as as an argument, then allocates some memory to point it to.

What is the correct way to efficiently (i.e. avoiding unnecessary copies) and safely (i.e. avoid memory leaks or segfaults) turn data from the C pointer into a `Vec`?

I've got something like:

```rust
extern "C" {
    // C function that allocates an array of floats
    fn allocate_data(data_ptr: *mut *const f32, data_len: *mut i32);
}

fn get_vec() -> Vec<f32> {
    // C will set this to length of array it allocates
    let mut data_len: i32 = 0;

    // C will point this at the array it allocates
    let mut data_ptr: *const f32 = std::ptr::null_mut();

    unsafe { allocate_data(&mut data_ptr, &mut data_len) };

    let data_slice = unsafe { slice::from_raw_parts(data_ptr as *const f32, data_len as usize) };
    data_slice.to_vec()
}
```

If I understand correctly, `.to_vec()` will copy data from the slice into a new `Vec`, so the underlying memory will still need to be freed (as the underlying memory for the slice won't be freed when it's dropped).

What is the correct approach for dealing with the above?

- can I create a `Vec` which takes ownership of the underlying memory, which is freed when the `Vec` is freed?
- if not, where/how in Rust should I free the memory that the C function allocated?
- anything else in the above that could/should be improved on?

answer

> can I create a `Vec` which takes ownership of the underlying memory, which is freed when the `Vec` is freed?

Not safely, no. You **must not** use `Vec::from_raw_parts` unless the pointer came from a `Vec` originally (well, from the same memory allocator). Otherwise, you will try to free memory that your allocator doesn't know about; a very bad idea.

Note that the same thing is true for `String::from_raw_parts`, as a `String` is a wrapper for a `Vec<u8>`.

> where/how in Rust should I free the memory that the C function allocated?

As soon as you are done with it and no sooner.

> anything else in the above that could/should be improved on?

- There's no need to cast the pointer when calling `slice::from_raw_parts`
- There's no need for explicit types on the variables
- Use `ptr::null`, not `ptr::null_mut`
- Perform a NULL pointer check
- Check the length is non-negative

```rust
use std::{ptr, slice};

extern "C" {
    fn allocate_data(data_ptr: *mut *const f32, data_len: *mut i32);
    fn deallocate_data(data_ptr: *const f32);
}

fn get_vec() -> Vec<f32> {
    let mut data_ptr = ptr::null();
    let mut data_len = 0;

    unsafe {
        allocate_data(&mut data_ptr, &mut data_len);
        assert!(!data_ptr.is_null());
        assert!(data_len >= 0);

        let v = slice::from_raw_parts(data_ptr, data_len as usize).to_vec();
        deallocate_data(data_ptr);

        v
    }
}

fn main() {}
```

------

You didn't state why you need it to be a `Vec`, but if you never need to change the size, you can create your own type that can be dereferenced as a slice and drops the data when appropriate:

```rust
use std::{ptr, slice};

extern "C" {
    fn allocate_data(data_ptr: *mut *const f32, data_len: *mut i32);
    fn deallocate_data(data_ptr: *const f32);
}

struct CVec {
    ptr: *const f32,
    len: usize,
}

impl std::ops::Deref for CVec {
    type Target = [f32];

    fn deref(&self) -> &[f32] {
        unsafe { slice::from_raw_parts(self.ptr, self.len) }
    }
}

impl Drop for CVec {
    fn drop(&mut self) {
        unsafe { deallocate_data(self.ptr) };
    }
}

fn get_vec() -> CVec {
    let mut ptr = ptr::null();
    let mut len = 0;

    unsafe {
        allocate_data(&mut ptr, &mut len);
        assert!(!ptr.is_null());
        assert!(len >= 0);

        CVec {
            ptr,
            len: len as usize,
        }
    }
}

fn main() {}
```


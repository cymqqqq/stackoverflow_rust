How to set a Rust array length dynamically?

Is it possible to create array with dynamic length?

No. By definition, arrays have a length *defined at compile time*. A variable (because it can *vary*) is not known at compile time. The compiler would not know how much space to allocate on the stack to provide storage for the array.

You will need to use a [`Vec`](https://doc.rust-lang.org/std/vec/struct.Vec.html):

```rust
let arr = vec![0; length];
```

You can create your own `HeapArray`. It's not that complicated if you read [`alloc`'s docs](https://doc.rust-lang.org/alloc/):

```rust
use std::alloc::{alloc, dealloc, Layout};
pub struct HeapArray<T> {
    ptr: *mut T,
    len: usize,
}

impl<T> HeapArray<T> {
    pub fn new(len: usize) -> Self {
        let ptr = unsafe {
            let layout = Layout::from_size_align_unchecked(len, std::mem::size_of::<T>());
            alloc(layout) as *mut T
        };
        Self { ptr, len }
    }
    pub fn get(&self, idx: usize) -> Option<&T> {
        if idx < self.len {
            unsafe { Some(&*(self.ptr.add(idx))) }
        } else {
            None
        }
    }
    pub fn get_mut(&self, idx: usize) -> Option<&mut T> {
        if idx < self.len {
            unsafe { Some(&mut *(self.ptr.add(idx))) }
        } else {
            None
        }
    }
    pub fn len(&self) -> usize {
        self.len
    }
}


impl<T> Drop for HeapArray<T> {
    fn drop(&mut self) {
        unsafe {
            dealloc(
                self.ptr as *mut u8,
                Layout::from_size_align_unchecked(self.len, std::mem::size_of::<T>()),
            )
        };
    }
}

impl<T> std::ops::Index<usize> for HeapArray<T> {
    type Output = T;
    fn index(&self, index: usize) -> &Self::Output {
        self.get(index).unwrap()
    }
}
impl<T> std::ops::IndexMut<usize> for HeapArray<T> {
    fn index_mut(&mut self, index: usize) -> &mut Self::Output {
        self.get_mut(index).unwrap()
    }
}
```

You may also add methods like `as_slice`, `get_unchecked`, etc.

Is it possible to control the size of an array using the type parameter of a generic?

Use *const generics*:

```rust
struct Vec<T: Sized, const COUNT: usize> {
    a: [T; COUNT],
}
```
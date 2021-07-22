In Rust, how do I create a mutable iterator? 

I'm having difficulty with lifetimes when trying to create a mutable iterator in safe Rust.

Here is what I have reduced my problem to:

```rust
struct DataStruct<T> {
    inner: Box<[T]>,
}

pub struct IterMut<'a, T> {
    obj: &'a mut DataStruct<T>,
    cursor: usize,
}

impl<T> DataStruct<T> {
    fn iter_mut(&mut self) -> IterMut<T> {
        IterMut { obj: self, cursor: 0 }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        let i = f(self.cursor);
        self.cursor += 1;
        self.obj.inner.get_mut(i)
    }
}

fn f(i: usize) -> usize {
   // some permutation of i
}
```

The structure of my `DataStruct` will never change, but I need to be able to mutate the contents of the elements stored within. For example,

```rust
let mut ds = DataStruct{ inner: vec![1,2,3].into_boxed_slice() };
for x in ds {
  *x += 1;
}
```

The compiler is giving me an error about conflicting lifetimes for the reference I am trying to return. The lifetime it finds that I am not expecting is the scope of the `next(&mut self)` function.

If I try to annotate the lifetime on `next()`, then the compiler, instead, tells me I haven't satisfied the Iterator trait. Is this solvable in safe rust?

Here is the error:

```none
error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/iter_mut.rs:25:24
   |
25 |         self.obj.inner.get_mut(i)
   |                        ^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 22:5...
  --> src/iter_mut.rs:22:5
   |
22 | /     fn next(&mut self) -> Option<Self::Item> {
23 | |         let i = self.cursor;
24 | |         self.cursor += 1;
25 | |         self.obj.inner.get_mut(i)
26 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/iter_mut.rs:25:9
   |
25 |         self.obj.inner.get_mut(i)
   |         ^^^^^^^^^^^^^^
note: but, the lifetime must be valid for the lifetime `'a` as defined on the impl at 19:6...
  --> src/iter_mut.rs:19:6
   |
19 | impl<'a, T> Iterator for IterMut<'a, T> {
   |      ^^
note: ...so that the types are compatible
  --> src/iter_mut.rs:22:46
   |
22 |       fn next(&mut self) -> Option<Self::Item> {
   |  ______________________________________________^
23 | |         let i = self.cursor;
24 | |         self.cursor += 1;
25 | |         self.obj.inner.get_mut(i)
26 | |     }
   | |_____^
   = note: expected  `std::iter::Iterator`
              found  `std::iter::Iterator`
```

answer1

The borrow checker is unable to prove that subsequent calls to `next()` won't access the same data. The reason why this is a problem is because the lifetime of the borrow is for the duration of the life of the iterator, so it can't prove that there won't be two mutable references to the same data at the same time.

There really isn't a way to solve this without unsafe code - or changing your data structures. You could do the equlivant of `slice::split_at_mut` but, given that you can't mutate the original data, you'd have to implement that in unsafe code anyway. An unsafe implementation could look something like this:

```rust
impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        let i = self.cursor;
        self.cursor += 1;
        if i < self.obj.inner.len() {
            let ptr = self.obj.inner.as_mut_ptr();
            unsafe {
                Some(&mut *ptr.add(i))
            }
        } else {
            None
        }
    }
}
```


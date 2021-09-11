Convert Option<&mut T> to *mut T

I'm writing a Rust wrapper around a C library and while doing so I'm trying to take advantage of the "nullable pointer optimization" mentioned in [The Book](https://doc.rust-lang.org/1.30.0/book/first-edition/ffi.html#the-nullable-pointer-optimization), but I can't find a good way to convert `Option<&T>` to `*const T` and `Option<&mut T>` to `*mut T` like what they're describing.

What I really want is to be able to call `Some(&foo) as *const _`. Unfortunately that doesn't work, so the next best thing I can think of is a trait on `Option<T>` that enables me to call `Some(&foo).as_ptr()`. The following code is a working definition and implementation for that trait:

```rust
use std::ptr;

trait AsPtr<T> {
    fn as_ptr(&self) -> *const T;
}

impl<'a, T> AsPtr<T> for Option<&'a T> {
    fn as_ptr(&self) -> *const T {
        match *self {
            Some(val) => val as *const _,
            None => ptr::null(),
        }
    }
}
```

Now that I can call `Some(&foo).as_ptr()` to get a `*const _`, I want to be able to call `Some(&mut foo).as_ptr()` to get a `*mut _`. The following is the new trait I made to do this:

```rust
trait AsMutPtr<T> {
    fn as_mut_ptr(&self) -> *mut T;
}

impl<'a, T> AsMutPtr<T> for Option<&'a mut T> {
    fn as_mut_ptr(&self) -> *mut T {
        match *self {
            Some(val) => val as *mut _,
            None => ptr::null_mut(),
        }
    }
}
```

The problem is, the `AsMutPtr` trait won't compile. When I try, I get the following error:

```none
error[E0507]: cannot move out of borrowed content
  --> src/lib.rs:22:15
   |
22 |         match *self {
   |               ^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider removing the `*`: `self`
23 |             Some(val) => val as *mut _,
   |                  --- data moved here
   |
note: move occurs because `val` has type `&mut T`, which does not implement the `Copy` trait
  --> src/lib.rs:23:18
   |
23 |             Some(val) => val as *mut _,
   |                  ^^^
```

I don't see what changed between the two traits that causes it to fail — I didn't think adding `mut` would make that big a difference. I tried adding a `ref`, but that just causes a different error, and I wouldn't expect to need that anyway.

Why doesn't the `AsMutPtr` trait work?

answer

Unfortunately, writing the trait impl for `&mut T` instead of `&T` *does* make a big difference. `&mut T`, as opposed to `&T`, is not `Copy`, therefore you cannot extract it out of a shared reference directly:

```none
& &T      --->  &T
& &mut T  -/->  &mut T
```

This is fairly natural - otherwise aliasing of mutable references would be possible, which violates Rust borrowing rules.

You may ask where that outer `&` comes from. It actually comes from `&self` in `as_mut_ptr()` method. If you have an immutable reference to something, even if that something contains mutable references inside it, you won't be able to use them to mutate the data behind them. This also would be a violation of borrowing semantics.

Unfortunately, I see no way to do this without unsafe. You need to have `&mut T` "by value" in order to cast it to `*mut T`, but you can't get it "by value" through a shared reference. Therefore, I suggest you to use `ptr::read()`:

```rust
use std::ptr;

impl<'a, T> AsMutPtr<T> for Option<&'a mut T> {
    fn as_mut_ptr(&self) -> *mut T {
        match *self {
            Some(ref val) => unsafe { ptr::read(val) as *mut _ },
            None => ptr::null_mut(),
        }
    }
}
```

`val` here is `& &mut T` because of `ref` qualifier in the pattern, therefore `ptr::read(val)` returns `&mut T`, aliasing the mutable reference. I think it is okay if it gets converted to a raw pointer immediately and does not leak out, but even though the result would be a raw pointer, it still means that you have two aliased mutable pointers. You should be very careful with what you do with them.

Alternatively, you may modify `AsMutPtr::as_mut_ptr()` to consume its target by value:

```rust
trait AsMutPtr<T> {
    fn as_mut_ptr(self) -> *mut T;
}

impl<'a, T> AsMutPtr<T> for Option<&'a mut T> {
    fn as_mut_ptr(self) -> *mut T {
        match self {
            Some(value) => value as *mut T,
            None => ptr::null_mut()
        }
    }
}
```

However, in this case `Option<&mut T>` will be consumed by `as_mut_ptr()`. This may not be feasible if, for example, this `Option<&mut T>` is stored in a structure. I'm not really sure whether it is possible to somehow perform reborrowing manually with `Option<&mut T>` as opposed to just `&mut T` (it won't be triggered automatically); if it is possible, then by-value `as_mut_ptr()` is probably the best overall solution.

answer

The problem is that you are reading an `&mut` out of an `&`, but `&mut`s are not `Copy` so must be moved - and you can't move out of a const reference. This actually explains Vladimir Matveev insight about `&&mut → &`s in terms of more fundamental properties.

This is actually relatively simply solved. If you can read a `*const _`, you can read a `*mut _`. The two are the same type, bar a flag that says "be careful, this is being shared". Since dereferences are unsafe either way, there's actually no reason to stop you casting between the two.

So you can actually do

```rust
match *self {
    Some(ref val) => val as *const _ as *mut _,
    None => ptr::null_mut(),
}
```

Read the immutable reference, make it an immutable pointer and then make it a mutable pointer. Plus it's all done through safe Rust so we know we're not breaking any aliasing rules.

That said, it's probably a really bad idea to actually use that `*mut` pointer until the `&mut` reference is gone. I would be very hesitant with this, and try to rethink your wrapper to something safer.

answer

Will this do what you expect?

```rust
trait AsMutPtr<T> {
    fn as_mut_ptr(self) -> *mut T;
}

impl<T> AsMutPtr<T> for Option<*mut T> {
    fn as_mut_ptr(self) -> *mut T {
        match self {
            Some(val) => val as *mut _,
            None => ptr::null_mut(),
        }
    }
}
```

answer

To avoid `unsafe` code, change the trait to accept `&mut self` instead of either `self` or `&self`:

```rust
trait AsMutPtr<T> {
    fn as_mut_ptr(&mut self) -> *mut T;
}

impl<'a, T> AsMutPtr<T> for Option<&'a mut T> {
    fn as_mut_ptr(&mut self) -> *mut T {
        match self {
            Some(v) => *v,
            None => ptr::null_mut(),
        }
    }
}
```

You could also reduce the implementation to one line, if you felt like it:

```rust
fn as_mut_ptr(&mut self) -> *mut T {
    self.as_mut().map_or_else(ptr::null_mut, |v| *v)
}
```

This can be used to give you multiple mutable raw pointers from the same source. This can easily lead you to causing mutable *aliasing*, so be careful:

```rust
fn example(mut v: Option<&mut u8>) {
    let b = v.as_mut_ptr();
    let a = v.as_mut_ptr();
}
```

I would recommend not converting the **immutable** reference to a **mutable** pointer as this is *very* likely to cause undefined behavior.


How to declare a generic type that is a raw pointer?

Given a struct that wraps a pointer,

```rust
pub struct Ptr<T> {
    ptr: T
}
```

Is it possible to declare that `T` must be a raw pointer type? eg `*mut SomeStruct` or `*const SomeStruct`.

Without this, I'm unable to perform operations like `&*self.ptr` within a method, since Rust doesn't know `ptr` can be treated like a pointer.

------

Note that this can be made to work:

```rust
pub struct Ptr<T> {
    ptr: *mut T
}
```

But in that case, it hard-codes `*mut`, where we might want `*const` in other cases.

answer

I'm not convinced this is worth doing, but if you're sure then you can just write a trait:

```rust
pub trait RawPtr: Sized {
    type Value;

    fn as_const(self) -> *const Self::Value {
        self.as_mut() as *const _
    }

    fn as_mut(self) -> *mut Self::Value {
        self.as_const() as *mut _
    }
}

impl<T> RawPtr for *const T {
    type Value = T;
    fn as_const(self) -> Self { self }
}

impl<T> RawPtr for *mut T {
    type Value = T;
    fn as_mut(self) -> Self { self }
}
```

Your can then require `P: RawPtr` when implementing functions:

```rust
pub struct Ptr<P> {
    ptr: P
}

impl<P: RawPtr> Ptr<P> {
    unsafe fn get(self) -> P::Value
        where P::Value: Copy
    {
        *self.ptr.as_const()
    }
}
```

Additionally, it's possible to define methods that are only available when `P` is a mutable pointer:

```rust
impl<T> Ptr<*mut T> {
    unsafe fn get_mut(&mut self) -> *mut T {
        self.ptr
    }
}
```


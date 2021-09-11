Convenient access to members of raw pointers?

The notation for accessing nested members of raw pointers for instances we know don't need to be checked against NULL can be rather awkward:

```rust
struct MyLink {
    link: *mut MyLink,
}
let var = *(*(*(*root).link).link).link;
```

Can raw pointer's struct members be accessed without having to explicitly de-reference each time? Maybe by using methods like `root.link().link().link()` or by wrapping the type?

While idiomatic Rust avoids this, there are exceptional cases where it isn't so easy to avoid. `Rc` has memory overhead, borrow checker causes problems for inter-linking members, C-API may require pointers... etc.

answer

If this is a recurring situation in your code, I'd just create a generic wrapper.

```rust
#[repr(C)]
#[derive(Hash)]
struct Ptr<T> {
    ptr: *mut T
}

impl<T> Ptr<T> {
    pub unsafe fn new(ptr: *mut T) -> Ptr<T> {
        debug_assert!(!ptr.is_null());
        Ptr { ptr: ptr }
    }

    #[inline(always)]
    pub fn as_pointer(&self) -> *mut T {
        self.ptr
    }
}

impl<T> Deref for Ptr<T> {
    type Target = T;

    #[inline(always)]
    fn deref(&self) -> &T {
        unsafe { &*self.ptr }
    }
}

impl<T> DerefMut for Ptr<T> {
    #[inline(always)]
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.ptr }
    }
}

impl<T> Copy for Ptr<T> { }
impl<T> Clone for Ptr<T> {
    #[inline(always)]
    fn clone(&self) -> Ptr<T> { *self }
}

impl<T> PartialEq for Ptr<T> {
    fn eq(&self, other: &Ptr<T>) -> bool {
        self.ptr == other.ptr
    }
}
```

We assert upon construction that the `ptr` is effectively not null, so we do not have to check again when dereferencing.

Then we let the language check for `Deref`/`DerefMut` when calling a method or accessing an attribute:

```rust
struct MyLink {
    link: Ptr<MyLink>,
}

fn main() {
    let mut link = MyLink { link: unsafe { Ptr::new(1 as *mut _) } };
    let next = MyLink { link: unsafe { Ptr::new(&mut link as *mut _) } };
    let _ = next.link;
}
```

answer

Wrapper methods can indeed improve readability of that code. Just by following [The Book](https://doc.rust-lang.org/book/raw-pointers.html#references-and-raw-pointers):

```rust
struct MyLink {
    link: *mut MyLink,
    pub n: i32,
}

impl MyLink {
    pub unsafe fn link(&self) -> &MyLink {
        &*self.link
    }

    pub unsafe fn mut_link(&mut self) -> &mut MyLink {
        &mut *self.link
    }
}
```

Whether to mark the method prototype as `unsafe` or not is up to your particular case, but the implementation must be in an unsafe block: fetching a reference from a pointer, even without dereferencing, is not safe.

Using it:

```rust
unsafe {
    let mut l1 = MyLink {
        link: 0 as *mut MyLink,
        n: 4,
    };

    let mut l2 = MyLink {
        link: &mut l1 as *mut MyLink,
        n: 3,
    };
    let n1 = l2.n;
    let n2 = l2.link().n;
    println!("{} -> {}", n1, n2);
}
```

[Gist](https://play.rust-lang.org/?gist=c6cf42d5f5c6a3a6c82b1d45acc3add7&version=stable&backtrace=0)

You can implement custom method for raw pointers in exactly the same way as for any other Rust type:

```rust
trait WickedRef<T>{
    unsafe fn wicked_ref<'x>(self) -> &'x T;
}

impl<T> WickedRef<T> for *mut T{
    unsafe fn wicked_ref<'x>(self) -> &'x T{
        &*self
    }    
}

root.link.wicked_ref().link.wicked_ref()
```


What is a “fat pointer”?

I've read the term "fat pointer" in several contexts already, but I'm not sure what exactly it means and when it is used in Rust. The pointer seems to be twice as large as a normal pointer, but I don't understand why. It also seems to have something to do with trait objects.

notes:

The term itself is not Rust-specific, BTW. *Fat pointer* generally refers to a pointer that stores some extra data besides just the address of the object being pointed to. If the pointer contains some *tag bits* and depending on those tag bits, the pointer sometimes isn't a pointer at all, it is called a *tagged pointer representation*. (E.g. on many Smalltalks VMs, pointers that end with a 1 bit are actually 31/63-bit integers, since pointers are word-aligned and thus never end in 1.) The HotSpot JVM calls its fat pointers *OOP*s (Object-Oriented Pointers). 

answer

**The term "fat pointer" is used to refer to references and raw pointers to \*dynamically sized types\* (DSTs) – slices or trait objects.** A fat pointer contains a pointer plus some information that makes the DST "complete" (e.g. the length).

Most commonly used types in Rust are *not* DSTs, but have a fixed size known at compile time. These types implement [the `Sized` trait](https://doc.rust-lang.org/stable/std/marker/trait.Sized.html). Even types that manage a heap buffer of dynamic size (like `Vec<T>`) are `Sized` as the compiler knows the exact number of bytes a `Vec<T>` instance will take up on the stack. There are currently four different kinds of DSTs in Rust.



## Slices (`[T]` and `str`)

The type `[T]` (for any `T`) is dynamically sized (so is the special "string slice" type `str`). That's why you usually only see it as `&[T]` or `&mut [T]`, i.e. behind a reference. This reference is a so-called "fat pointer". Let's check:

```rust
dbg!(size_of::<&u32>());
dbg!(size_of::<&[u32; 2]>());
dbg!(size_of::<&[u32]>());
```

This prints (with some cleanup):

```none
size_of::<&u32>()      = 8
size_of::<&[u32; 2]>() = 8
size_of::<&[u32]>()    = 16
```

So we see that a reference to a normal type like `u32` is 8 bytes large, as is a reference to an array `[u32; 2]`. Those two types are not DSTs. But as `[u32]` is a DST, the reference to it is twice as large. **In the case of slices, the additional data that "completes" the DST is simply the length.** So one could say the representation of `&[u32]` is something like this:

```rust
struct SliceRef { 
    ptr: *const u32, 
    len: usize,
}
```



## Trait objects (`dyn Trait`)

When using traits as trait objects (i.e. type erased, dynamically dispatched), these trait objects are DSTs. Example:

```rust
trait Animal {
    fn speak(&self);
}

struct Cat;
impl Animal for Cat {
    fn speak(&self) {
        println!("meow");
    }
}

dbg!(size_of::<&Cat>());
dbg!(size_of::<&dyn Animal>());
```

This prints (with some cleanup):

```none
size_of::<&Cat>()        = 8
size_of::<&dyn Animal>() = 16
```

Again, `&Cat` is only 8 bytes large because `Cat` is a normal type. But `dyn Animal` is a trait object and therefore dynamically sized. As such, `&dyn Animal` is 16 bytes large.

**In the case of trait objects, the additional data that completes the DST is a pointer to the vtable (the vptr).** I cannot fully explain the concept of vtables and vptrs here, but they are used to call the correct method implementation in this virtual dispatch context. The vtable is a static piece of data that basically only contains a function pointer for each method. With that, a reference to a trait object is basically represented as:

```rust
struct TraitObjectRef {
    data_ptr: *const (),
    vptr: *const (),
}
```

(This is different from C++, where the vptr for abstract classes is stored within the object. Both approaches have advantages and disadvantages.)



## Custom DSTs

It's actually possible to create your own DSTs by having a struct where the last field is a DST. This is rather rare, though. One prominent example is `std::path::Path`.

A reference or pointer to the custom DST is also a fat pointer. The additional data depends on the kind of DST inside the struct.



## Exception: Extern types

In [RFC 1861](https://github.com/rust-lang/rfcs/blob/master/text/1861-extern-types.md), the `extern type` feature was introduced. Extern types are also DSTs, but pointers to them are *not* fat pointers. Or more exactly, as the RFC puts it:

> In Rust, pointers to DSTs carry metadata about the object being pointed to. For strings and slices this is the length of the buffer, for trait objects this is the object's vtable. For extern types the metadata is simply `()`. This means that a pointer to an extern type has the same size as a `usize` (ie. it is not a "fat pointer").

But if you are not interacting with a C interface, you probably won't ever have to deal with these extern types.





------

Above, we've seen the sizes for immutable references. Fat pointers work the same for mutable references, immutable raw pointers and mutable raw pointers:

```rust
size_of::<&[u32]>()       = 16
size_of::<&mut [u32]>()   = 16
size_of::<*const [u32]>() = 16
size_of::<*mut [u32]>()   = 16
```
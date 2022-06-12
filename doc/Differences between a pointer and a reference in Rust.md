Differences between a pointer and a reference in Rust

A pointer * and a reference & in Rust share the same representation (they both represent the memory address of a piece of data).

What's the practical differences when writing code though?

When porting C++ code to Rust can they be replaced safely (c++ pointer --> rust pointer, c++ reference --> rust reference ) ?

notes:

Broadly speaking you can do more with references in Rust than in C++, and raw pointers in C++ are used for a lot more than in Rust. If you are using modern C++ with mostly smart pointers those can often be translated directly (e.g. `std::unique_ptr` -> `Box`, `std::shared_ptr` -> `Arc`, although not all of them are so straightforward). If your C++ has a lot of C pointers in it you will need to evaluate how they are being used. A lot of C++ code uses raw pointers unnecessarily where references or smart pointers would work better.

answer

Use references when you can, use pointers when you must. If you're not doing FFI or memory management beyond what the compiler can validate, you don't need to use pointers.

Both references and pointers exist in two variants. There are shared references `&` and mutable references `&mut`. There are const pointers `*const` and mut pointers `*mut` (which map to const and non-const pointers in C). However, the semantics of references is completely different from the semantics of pointers.

References are generic over a type and over a lifetime. Shared references are written `&'a T` in long form (where `'a` and `T` are parameters). The lifetime parameter can be [omitted](https://doc.rust-lang.org/nomicon/lifetime-elision.html) in many situations. The lifetime parameter is used by the compiler to ensure that a reference doesn't live longer than the borrow is valid for.

Pointers have no lifetime parameter. Therefore, the compiler cannot check that a particular pointer is valid to use. That's why dereferencing a pointer is considered `unsafe`.

When you create a shared reference to an object, that *freezes* the object (i.e. the object becomes immutable while the shared reference exists), unless the object uses some form of interior mutability (e.g. using `Cell`, `RefCell`, `Mutex` or `RwLock`). However, when you have a const pointer to an object, that object may still change while the pointer is alive.

When you have a mutable reference to an object, you are *guaranteed* to have exclusive access to that object through this reference. Any other way to access the object is either disabled temporarily or impossible to achieve. For example:

```rust
let mut x = 0;
{
    let y = &mut x;
    let z = &mut x; // ERROR: x is already borrowed mutably
    *y = 1; // OK
    x = 2; // ERROR: x is borrowed
}
x = 3; // OK, y went out of scope
```

Mut pointers have no such guarantee.

A reference cannot be null (much like C++ references). A pointer can be null.

Pointers may contain any numerical value that could fit in a `usize`. Initializing a pointer is not `unsafe`; only dereferencing it is. On the other hand, producing an invalid reference is considered [undefined behavior](https://doc.rust-lang.org/reference/behavior-considered-undefined.html), even if you never dereference it.

If you have a `*const T`, you can freely cast it to a `*const U` or to a `*mut T` using `as`. You can't do that with references. However, you can cast a reference to a pointer using `as`, and you can "upgrade" a pointer to a reference by dereferencing the pointer (which, again, is `unsafe`) and then borrowing the place using `&` or `&mut`. For example:

```rust
use std::ffi::OsStr;
use std::path::Path;

pub fn os_str_to_path(s: &OsStr) -> &Path {
    unsafe { &*(s as *const OsStr as *const Path) }
}
```

In C++, references are "automatically dereferenced pointers". In Rust, you often still need to dereference references explicitly. The exception is when you use the `.` operator: if the left side is a reference, the compiler will automatically dereference it (recursively if necessary!). Pointers, however, are not automatically dereferenced. This means that if you want to dereference and access a field or a method, you need to write `(*pointer).field` or `(*pointer).method()`. There is no `->` operator in Rust.
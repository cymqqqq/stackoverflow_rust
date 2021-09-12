Understanding and relationship between Box, ref, & and *

I'm a bit confused about how pointers work in Rust. There's `ref`, `Box`, `&`, `*`, and I'm not sure how they work together.

Here's how I understand it currently:

1. `Box` isn't really a pointer - it's a way to allocate data on the heap, and pass around unsized types (traits especially) in function arguments.

2. `ref` is used in pattern matching to borrow something that you match on, instead of taking it. For example,

   ```rust
   let thing: Option<i32> = Some(4);
   match thing {
       None => println!("none!"),
       Some(ref x) => println!("{}", x), // x is a borrowed thing
   }
   println!("{}", x + 1); // wouldn't work without the ref since the block would have taken ownership of the data
   ```

3. `&` is used to make a borrow (borrowed pointer). If I have a function `fn foo(&self)` then I'm taking a reference to myself that will expire after the function terminates, leaving the caller's data alone. I can also pass data that I want to retain ownership of by doing `bar(&mydata)`.

4. `*` is used to make a raw pointer: for example, `let y: i32 = 4; let x = &y as *const i32`. I understand pointers in C/C++ but I'm not sure how this works with Rust's type system, and how they can be safely used. I'm also not sure what the use cases are for this type of pointer. Additionally, the `*` symbol can be used to dereference things (what things, and why?).

Could someone explain the 4th type of pointer to me, and verify that my understanding of the other types is correct? I'd also appreciate anyone pointing out any common use cases that I haven't mentioned.

answer

First of all, all of the items you listed are really different things, even if they are related to pointers. `Box` is a library-defined smart pointer type; `ref` is a syntax for pattern matching; `&` is a reference operator, doubling as a sigil in reference types; `*` is a dereference operator, doubling as a sigil in raw pointer types. See below for more explanation.

There are four basic pointer types in Rust which can be divided in two groups - references and raw pointers:

```rust
&T        - immutable (shared) reference
&mut T    - mutable (exclusive) reference

*const T  - immutable raw pointer
*mut T    - mutable raw pointer
```

The difference between the last two is very thin, because either can be cast to another without any restrictions, so `const`/`mut` distinction there serves mostly as a lint. Raw pointers can be created freely to anything, and they also can be created out of thin air from integers, for example.

Naturally, this is not so for references - reference types and their interaction define one of the key feature of Rust: borrowing. References have a lot of restrictions on how and when they could be created, how they could be used and how they interact with each other. In return, they can be used without `unsafe` blocks. What borrowing is exactly and how it works is out of scope of this answer, though.

Both references and raw pointers can be created using `&` operator:

```rust
let x: u32 = 12;

let ref1: &u32 = &x;
let raw1: *const u32 = &x;

let ref2: &mut u32 = &mut x;
let raw2: *mut u32 = &mut x;
```

Both references and raw pointers can be dereferenced using `*` operator, though for raw pointers it requires an `unsafe` block:

```rust
*ref1; *ref2;

unsafe { *raw1; *raw2; }
```

The dereference operator is often omitted, because another operator, the "dot" operator (i.e., `.`), automatically references or dereferences its left argument. So, for example, if we have these definitions:

```rust
struct X { n: u32 };

impl X {
    fn method(&self) -> u32 { self.n }
}
```

then, despite that `method()` takes `self` by reference, `self.n` automatically dereferences it, so you won't have to type `(*self).n`. Similar thing happens when `method()` is called:

```rust
let x = X { n: 12 };
let n = x.method();
```

Here, the compiler automatically references `x` in `x.method()`, so you won't have to write `(&x).method()`.

The next to last piece of code also demonstrated the special `&self` syntax. It means just `self: &Self`, or, more specifically, `self: &X` in this example. `&mut self`, `*const self`, `*mut self` also work.

So, references are the main pointer kind in Rust and should be used almost always. Raw pointers, which don't have restrictions of references, should be used in low-level code implementing high-level abstractions (collections, smart pointers, etc.) and in FFI (interacting with C libraries).

Rust also has [dynamically-sized (or unsized) types](http://doc.rust-lang.org/book/unsized-types.html). These types do not have a definite statically-known size and therefore can only be used through a pointer/reference. However, only a pointer is not enough - additional information is needed, for example, length for slices or a pointer to a virtual methods table for trait objects. This information is "embedded" in pointers to unsized types, making these pointers "fat".

A fat pointer is basically a structure which contains the actual pointer to the piece of data and some additional information (length for slices, pointer to vtable for trait objects). What's important here is that Rust handles these details about pointer contents absolutely transparently for the user - if you pass `&[u32]` or `*mut SomeTrait` values around, corresponding internal information will be automatically passed along.

`Box<T>` is one of the smart pointers in the Rust standard library. It provides a way to allocate enough memory on the heap to store a value of the corresponding type, and then it serves as a handle, a pointer to that memory. `Box<T>` owns the data it points to; when it is dropped, the corresponding piece of memory on the heap is deallocated.

A very useful way to think of boxes is to consider them as regular values, but with a fixed size. That is, `Box<T>` is equivalent to just `T`, except it always takes a number of bytes which correspond to the pointer size of your machine. We say that (owned) boxes provide *value semantics*. Internally, they are implemented using raw pointers, like almost any other high-level abstraction.

`Box`es (in fact, this is true for almost all of the other smart pointers, like `Rc`) can also be borrowed: you can get a `&T` out of `Box<T>`. This can happen automatically with the `.` operator or you can do it explicitly by dereferencing and referencing it again:

```rust
let x: Box<u32> = Box::new(12);
let y: &u32 = &*x;
```

In this regard, `Box`es are similar to built-in pointers - you can use dereference operator to reach their contents. This is possible because the dereference operator in Rust is overloadable, and it is overloaded for most (if not all) of the smart pointer types. This allows easy borrowing of these pointers contents.

And, finally, `ref` is just a syntax in patterns to obtain a variable of the reference type instead of a value. For example:

```rust
let x: u32 = 12;

let y = x;           // y: u32, a copy of x
let ref z = x;       // z: &u32, points to x
let ref mut zz = x;  // zz: &mut u32, points to x
```

While the above example can be rewritten with reference operators:

```rust
let z = &x;
let zz = &mut x;
```

(which would also make it more idiomatic), there are cases when `ref`s are indispensable, for example, when taking references into enum variants:

```rust
let x: Option<Vec<u32>> = ...;

match x {
    Some(ref v) => ...
    None => ...
}
```

In the above example, `x` is only borrowed inside the whole `match` statement, which allows using `x` after this `match`. If we write it as such:

```rust
match x {
    Some(v) => ...
    None => ...
}
```

then `x` will be consumed by this `match` and will become unusable after it.

answer

`Box` is logically a newtype around a raw pointer (`*const T`). However, it allocates and deallocates its data during construction and destruction, so does not have to borrow data from some other source.

The same thing is true of other pointer types, like `Rc` - a reference counted pointer. These are structs containing private raw pointers which they allocate into and deallocate from.

A raw pointer has exactly the same layout as a a normal pointer, so are not compatible with C pointers in several cases. Importantly, `*const str` and `*const [T]` are *fat pointers*, which means they contain extra information about the value's length.

However, raw pointers makes absolutely no guarantees as to their validity. For example, I can safely do

```rust
123 as *const String
```

This pointer is invalid, since the memory location `123` does not point to a valid `String`. Thus, when dereferencing one, an `unsafe` block is required.

Further, whereas borrows are required to respect certain laws - namely that you cannot have multiple borrows if one is mutable - raw pointers do not have to respect this. [There are other, weaker, laws that must be obeyed,](http://llvm.org/docs/LangRef.html#pointer-aliasing-rules) but you're less likely to run afoul of these.

There is no logical difference between `*mut` and `*const`, although they may need to be casted to the other to do certain operations - the difference is documentative.

answer

References and raw pointers are the same thing at the implementation level. The difference from the programmer perspective is that references are safe (in Rust terms), but raw pointers are not.

The borrow checker guarantees that references are always valid (lifetime management), that you can have only one mutable reference at time, etc.

These type of constraint can be too strict for many use cases, so raw pointers (which do not have any constraints, like in C/C++) are useful to implement low-level data structures, and in general low-level stuff. However, you can only dereference raw pointers or do operations on them inside an `unsafe` block.

The containers in the standard library are implemented using raw pointers, `Box` and `Rc` too.

`Box` and `Rc` are what smart pointers are in C++, that is wrappers around raw pointers.


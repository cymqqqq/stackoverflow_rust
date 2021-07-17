How to provide an implementation of a generic struct in Rust?

I have a struct `MyStruct` that takes a generic parameter `T: SomeTrait`, and I want to implement a `new` method for `MyStruct`. This works:

```rust
/// Constraint for the type parameter `T` in MyStruct
pub trait SomeTrait: Clone {}

/// The struct that I want to construct with `new`
pub struct MyStruct<T: SomeTrait> {
    value: T,
}

fn new<T: SomeTrait>(t: T) -> MyStruct<T> {
    MyStruct { value: t }
}

fn main() {}
```

I wanted to put the `new` function inside an `impl` block like this:

```rust
impl MyStruct {
    fn new<T: SomeTrait>(t: T) -> MyStruct<T> {
        MyStruct { value: t }
    }
}
```

But that fails to compile with:

```rust
error[E0107]: wrong number of type arguments: expected 1, found 0
 --> src/main.rs:9:6
  |
9 | impl MyStruct {
  |      ^^^^^^^^ expected 1 type argument
```

If I try to put it like this:

```rust
impl MyStruct<T> {
    fn new(t: T) -> MyStruct<T> {
        MyStruct { value: t }
    }
}
```

The error changes to:

```rust
error[E0412]: cannot find type `T` in this scope
 --> src/main.rs:9:15
  |
9 | impl MyStruct<T> {
  |               ^ not found in this scope
```

How do I provide an implementation of a generic struct? Where do I put the generic parameters and their constraints?

answer1

The type parameter `<T: SomeTrait>` should come right after the `impl` keyword:

```rust
impl<T: SomeTrait> MyStruct<T> {
    fn new(t: T) -> Self {
        MyStruct { value: t }
    }
}
```

If the list of types and constraints in `impl<...>` becomes too long, you can use the `where`-syntax and list the constraints separately:

```rust
impl<T> MyStruct<T>
where
    T: SomeTrait,
{
    fn new(t: T) -> Self {
        MyStruct { value: t }
    }
}
```

Note the usage of `Self`, which is a shortcut for `MyStruct<T>` available inside of the `impl` block.

------

**Remarks**

1. The reason why `impl<T>` is required is explained in [this answer](https://stackoverflow.com/a/45473717/2707792). Essentially, it boils down to the fact that both `impl<T> MyStruct<T>` and `impl MyStruct<T>` are valid, but mean different things.

2. When you move `new` into the `impl` block, you should remove the superfluous type parameters, otherwise the interface of your struct will become unusable, as the following example shows:

   ```rust
   // trait SomeTrait and struct MyStruct as above
   // [...]
   
   impl<T> MyStruct<T>
   where
       T: SomeTrait,
   {
       fn new<S: SomeTrait>(t: S) -> MyStruct<S> {
           MyStruct { value: t }
       }
   }
   
   impl SomeTrait for u64 {}
   impl SomeTrait for u128 {}
   
   fn main() {
       // just a demo of problematic code, don't do this!
       let a: MyStruct<u128> = MyStruct::<u64>::new::<u128>(1234);
       //                                 ^
       //                                 |
       //        This is an irrelevant type
       //        that cannot be inferred. Not only will the compiler
       //        force you to provide an irrelevant type, it will also
       //        not prevent you from passing incoherent junk as type
       //        argument, as this example demonstrates. This happens 
       //        because `S` and `T` are completely unrelated.
   }
   ```


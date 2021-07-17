How do I implement a trait with a generic method?

I'm trying to implement a trait which contains a generic method.

```rust
trait Trait {
    fn method<T>(&self) -> T;
}

struct Struct;

impl Trait for Struct {
    fn method(&self) -> u8 {
        return 16u8;
    }
}
```

I get:

```none
error[E0049]: method `method` has 0 type parameters but its trait declaration has 1 type parameter
 --> src/lib.rs:8:5
  |
2 |     fn method<T>(&self) -> T;
  |     ------------------------- expected 1 type parameter
...
8 |     fn method(&self) -> u8 {
  |     ^^^^^^^^^^^^^^^^^^^^^^ found 0 type parameters
```

How should I write the `impl` block correctly?

answer1

Type parameters in functions and methods are *universal*. This means that for all trait implementers, `Trait::method<T>` must be implemented for any `T` with the exact same constraints as those indicated by the trait (in this case, the constraint on `T` is only the implicit `Sized`).

The compiler's error message that you indicated suggests that it was still expecting the parameter type `T`. Instead, your `Struct` implementation is assuming that `T = u8`, which is incorrect. The type parameter is decided by the caller of the method rather than the implementer, so `T` might not always be `u8`.

If you wish to let the implementer choose a specific type, that has to be materialized in an associated type instead.

```rust
trait Trait {
    type Output;

    fn method(&self) -> Self::Output;
}

struct Struct;

impl Trait for Struct {
    type Output = u8;

    fn method(&self) -> u8 {
        16
    }
}
```

Read also this section of *The Rust Programming Language*: [Specifying placeholder types in trait definitions with associated types](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types).

answer2

In addition to the method using an associated type, from [this answer](https://stackoverflow.com/a/53085395), you can also add the generic to the trait.

```rust
trait Trait<T> {
    fn method(&self) -> T;
}

impl Trait<u8> for Struct {
    fn method(&self) -> u8 {
        16
    }
}
```

You use the "associated type" way when there is only one logical form of the trait to use. You can use the generic trait when there is more than one output type that makes sense, for example this is legal:

```rust
struct Struct;

trait Trait<T> {
    fn method(&self) -> T;
}

impl Trait<u8> for Struct {
    fn method(&self) -> u8 {
        16
    }
}

impl Trait<String> for Struct {
    fn method(&self) -> String {
        "hello".to_string()
    }
}

fn main() {
    let s = Struct;
    let a: u8 = s.method();
    let b: String = s.method();
    println!("a={}, b={}", a, b);
}
```


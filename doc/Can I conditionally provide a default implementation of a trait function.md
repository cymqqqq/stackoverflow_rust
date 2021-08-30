Can I conditionally provide a default implementation of a trait function?

I have the following trait:

```rust
trait MyTrait {
    type A;
    type B;

    fn foo(a: Self::A) -> Self::B;

    fn bar(&self);
}
```

There are other functions like `bar` that must be always implemented by the user of the trait.

I would like to give `foo` a default implementation, but only when the type `A = B`.

Pseudo-Rust code:

```rust
impl??? MyTrait where Self::A = Self::B ??? {
    fn foo(a: Self::A) -> Self::B {
        a
    }
}
```

This would be possible:

```rust
struct S1 {}

impl MyTrait for S1 {
    type A = u32;
    type B = f32;

    // `A` is different from `B`, so I have to implement `foo`
    fn foo(a: u32) -> f32 {
        a as f32
    }

    fn bar(&self) {
        println!("S1::bar");
    }
}

struct S2 {}

impl MyTrait for S2 {
    type A = u32;
    type B = u32;

    // `A` is the same as `B`, so I don't have to implement `foo`,
    // it uses the default impl

    fn bar(&self) {
        println!("S2::bar");
    }
}
```

Is that possible in Rust?

answer1

You can provide a default implementation in the trait definition itself by introducing a redundant type parameter:

```rust
trait MyTrait {
    type A;
    type B;

    fn foo<T>(a: Self::A) -> Self::B
    where
        Self: MyTrait<A = T, B = T>,
    {
        a
    }
}
```

This default implementation can be overridden for individual types. However, the specialized versions will inherit the trait bound from the definition of `foo()` on the trait so you can only actually *call* the method if `A == B`:

```rust
struct S1;

impl MyTrait for S1 {
    type A = u32;
    type B = f32;

    fn foo<T>(a: Self::A) -> Self::B {
        a as f32
    }
}

struct S2;

impl MyTrait for S2 {
    type A = u32;
    type B = u32;
}

fn main() {
    S1::foo(42);  // Fails with compiler error
    S2::foo(42);  // Works fine
}
```

Rust also has an [unstable impl specialization feature](https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md), but I don't think it can be used to achieve what you want.

answer2

Will this suffice?:

```rust
trait MyTrait {
    type A;
    type B;

    fn foo(a: Self::A) -> Self::B;
}

trait MyTraitId {
    type AB;
}

impl<P> MyTrait for P
where
    P: MyTraitId
{
    type A = P::AB;
    type B = P::AB;

    fn foo(a: Self::A) -> Self::B {
        a
    }
}

struct S2;

impl MyTraitId for S2 {
    type AB = i32;
}
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=3128566f15b640154ba0305eba7d65df)

As noted, it'll bump into problems if `MyTrait` as other methods that `MyTraitId` can't provide an implementation for.


Expected type parameter, found u8, but the type parameter is u8

```rust
trait Foo {
    fn foo<T>(&self) -> T;
}

struct Bar {
    b: u8,
}

impl Foo for Bar {
    fn foo<u8>(&self) -> u8 {
        self.b
    }
}

fn main() {
    let bar = Bar {
        b: 2,
    };
    println!("{:?}", bar.foo());
}
```

([Playground](https://play.rust-lang.org/?gist=fa601bca89894e0150306a2e1b57841a&version=stable&backtrace=0))

The above code results in the following error:

```none
error[E0308]: mismatched types
  --> <anon>:11:9
   |
11 |         self.b
   |         ^^^^^^ expected type parameter, found u8
   |
   = note: expected type `u8` (type parameter)
              found type `u8` (u8)
```

My guess is, the problem comes from generic function in trait.

answer1

The following code does not do what you expect

```rust
impl Foo for Bar {
    fn foo<u8>(&self) -> u8 {
        self.b
    }
}
```

It introduces a generic type called `u8` which shadows the concrete type `u8`. Your function would be 100% the same as

```rust
impl Foo for Bar {
    fn foo<T>(&self) -> T {
        self.b
    }
}
```

Which cannot work in this case because `T`, chosen by the *caller* of `foo`, isn't guaranteed to be `u8`.

To solve this problem in general, choose generic type names that do not conflict with concrete type names. Remember that the function signature in the implementation *has* to match the signature in the trait definition.

------

To solve the issue presented, where you wish to *fix* the generic type as a specific value, you can move the generic parameter to the trait, and implement the trait just for `u8`:

```rust
trait Foo<T> {
    fn foo(&self) -> T;
}

struct Bar {
    b: u8,
}

impl Foo<u8> for Bar {
    fn foo(&self) -> u8 {
        self.b
    }
}
```

Or you can use an associated trait, if you never want multiple `Foo` impls for a specific type (thanks @MatthieuM):

```rust
trait Foo {
    type T;
    fn foo(&self) -> T;
}

struct Bar {
    b: u8,
}

impl Foo for Bar {
    type T = u8;
    fn foo(&self) -> u8 {
        self.b
    }
}
```

answer2

Let's look at a slightly more general example. We will define a trait with a function that takes and returns a generic type:

```rust
trait Foo {
    fn foo<T>(&self, value: T) -> T;
}

struct Bar;

impl Foo for Bar {
    fn foo<u8>(&self, value: u8) -> u8 {
        value
    }

    // Equivalent to 
    // fn foo<T>(&self, value: T) -> T {
    //    value
    // }
}
```

As [oli_obk - ker has already explained](https://stackoverflow.com/a/37410775/155423), `fn foo<u8>(&self, value: u8) -> u8` defines a *generic type parameter* called `u8` which shadows the built in type `u8`. This is allowed for forwards compatibility reasons — what if you decided to call your generic type `Fuzzy` and then a crate (or the standard library!) introduced a type also called `Fuzzy`? If shadowing wasn't allowed, your code would stop compiling!

However, you should *avoid* using generic type parameters that are existing types - as shown, it's just confusing.

------

Many times, people fall into this trap because they are trying to specify a concrete type for a generic parameter. This shows that there is a misunderstanding of how generic types work: generic types are **chosen by the caller of the function**; the implementation doesn't get to pick what they are!

The solutions outlined in the [other answer](https://stackoverflow.com/a/37410775/155423) remove the ability for the caller to choose the generic.

- Moving the generic type to the trait and only implementing it for a handful of types means that there's only a small set of implementations available. If the caller tries to use a type that doesn't have a corresponding implementation, it will just fail to compile.
- Choosing an associated type is *designed* to allow the implementor of the trait to choose the type, and is often the correct solution in this case.

See [When is it appropriate to use an associated type versus a generic type?](https://stackoverflow.com/q/32059370/155423) for more details on picking between the two.

This problem is more common for people learning Rust that haven't yet internalized what the various syntaxes of generics do. A quick refresher...

Functions / methods:

```rust
fn foo<T>(a: T) -> T
//    ^-^ declares a generic type parameter

fn foo<T>(a: T) -> T
//           ^     ^ a type, which can use a previously declared parameter
```

Traits:

```rust
trait Foo<T>
//       ^-^ declares a generic type parameter
```

Structs / enums:

```rust
enum Wuuf<T>
//       ^-^ declares a generic type parameter

struct Quux<T>
//         ^-^ declares a generic type parameter
```

Implementations:

```rust
impl<T> Foo<T> for Bar<T>
//  ^-^ declares a generic type parameter

impl<T> Foo<T> for Bar<T>
//      ^----^     ^----^ a type, which can use a previously declared parameter
```

In these examples, only a *type* is allowed to specify a concrete type, which is why `impl Foo<u8> for Bar` makes sense. You could get back into the original situation by declaring a generic called `u8` on a trait too!

```rust
impl<u8> Foo<u8> for Bar // Right back where we started
```

------

There's another, rarer case: you didn't mean for the type to be generic in any fashion in the first place! If that's the case, rename or remove the generic type declaration and write a standard function. For our example, if we always wanted to return a `u8`, regardless of what type was passed in, we'd just write that:

```rust
trait Foo {
    fn foo<T>(&self, value: T) -> u8;
}

impl Foo for Bar {
    fn foo<T>(&self, value: T) -> u8 {
        42
    }
}
```

------

Good news! At least as of Rust 1.17, introducing this type of error is a bit easier to spot thanks to a warning:

```none
warning: type parameter `u8` should have a camel case name such as `U8`
```


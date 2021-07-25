Rust box a single trait that encompasses generic traits

I am in a situation where I want some objects to implement a trait, say "Base", and some other objects will implements a trait "Super". The Super trait also has to be generic for `T : Base` so that I can automatically implement parts of Super based on which Base it was specialized with. Now this seems to work fine with the following trivial example

```rust
trait Base {
    fn say_hi() -> &'static str;
}

struct BaseOne {}
struct BaseTwo {}

impl Base for BaseOne {
    fn say_hi() -> &'static str {
        "hi!"
    }
}

impl Base for BaseTwo {
    fn say_hi() -> &'static str {
        "hello!"
    }
}

trait Super<T: Base> {
    fn say_hi(&self) -> &'static str {
        T::say_hi()
    }
}

struct SuperOne;
struct SuperTwo;

impl Super<BaseOne> for SuperOne {}
impl Super<BaseTwo> for SuperTwo {}
```

My problem comes in with my next requirement, which is that I want to be able to store a vector of objects which implement Super, regardless of which Base it is specialized for. My idea for this is to create a trait that covers all Supers, such as the AnySuper trait below

```rust
trait AnySuper {
    fn say_hi(&self) -> &'static str;
}

impl<T> AnySuper for dyn Super<T> where T : Base {
    fn say_hi(&self) -> &'static str {
        Super::say_hi(self)
    }
}
```

And then store a vector of Box such as in the example below

```rust
fn main() {
    let one = Box::new(SuperOne);
    let two = Box::new(SuperTwo);
    
    let my_vec: Vec<Box<dyn AnySuper>> = Vec::new();
    
    my_vec.push(one);
    my_vec.push(two);
}
```

But unfortunately that fails with the following error

```rust
error[E0277]: the trait bound `SuperOne: AnySuper` is not satisfied
  --> src/main.rs:52:17
   |
52 |     my_vec.push(one);
   |                 ^^^ the trait `AnySuper` is not implemented for `SuperOne`
   |
   = note: required for the cast to the object type `dyn AnySuper`

error[E0277]: the trait bound `SuperTwo: AnySuper` is not satisfied
  --> src/main.rs:53:17
   |
53 |     my_vec.push(two);
   |                 ^^^ the trait `AnySuper` is not implemented for `SuperTwo`
   |
   = note: required for the cast to the object type `dyn AnySuper`
```

Which is a bit strange because in my mind I have implemented AnySuper for all `Super<T>`. So my question is, am I doing something fundamentally wrong or is there just an issue with my syntax?

P.S. I have set up a playground with this code at https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=a54e3e9044f1edaeb24d8ad934eaf7ec if anyone wants to play around with it.

answer1

I think something strange is happening since you're going through two layers of trait object indirection, i.e. from the concrete `Box<SuperOne>` to `Box<dyn Super<BaseOne>>` to `Box<dyn AnySuper>`. Certainly, Rust is equipped to handle the first case, as we do that all the time, but the second is not something I've seen.

From the sound of it, though, you want to say "`AnySuper` is implemented whenever `Super` is for any `T`", and what you've written in the code is "`AnySuper` is implemented for this one funny trait object type called `dyn Super<T>`". Let's try to write an actual blanket implementation.

```rust
impl<S, T> AnySuper for S where S : Super<T>, T : Base
```

But now we get some rather exciting error messages about unconstrained types.

```rust
error[E0207]: the type parameter `T` is not constrained by the impl trait, self type, or predicates
  --> src/main.rs:36:9
   |
36 | impl<S, T> AnySuper for S where S : Super<T>, T : Base {
   |         ^ unconstrained type parameter
```

You can read more about the motivation for this error on [the relevant RFC](https://github.com/rust-lang/rfcs/blob/master/text/0447-no-unused-impl-parameters.md), but the bottom line is that Rust can't figure out what `T` should be for some arbitrary `S`.

And that's true in general. If you give Rust some arbitrary thing that implements `Super<T>`, it might implement `Super<T>` for several different `T`, and then `AnySuper` would have to choose between those, which it can't do. Instead, we need to promise Rust that, for a given implementor, there's only one possible `T`, and we can do that by making `T` an [associated type](https://doc.rust-lang.org/reference/items/associated-items.html#associated-types).

```rust
trait Super {
    type T : Base;
    fn say_hi(&self) -> &'static str {
        Self::T::say_hi()
    }
}

impl Super for SuperOne {
    type T = BaseOne;
}
impl Super for SuperTwo {
    type T = BaseTwo;
}

impl<S> AnySuper for S where S : Super, S::T : Base {
    fn say_hi(&self) -> &'static str {
        Super::say_hi(self)
    }
}
```

Now Rust will happily accept your vector of `AnySuper`.

[Rust Playground link](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=3f08cb2f7cb06ae9000c5c038446bc12)

Working code example:

```rust
trait Base {
    fn say_hi() -> &'static str;
}

struct BaseOne {}
struct BaseTwo {}

impl Base for BaseOne {
    fn say_hi() -> &'static str {
        "hi!"
    }
}

impl Base for BaseTwo {
    fn say_hi() -> &'static str {
        "hello!"
    }
}

trait Super {
    type T : Base;
    fn say_hi(&self) -> &'static str {
        Self::T::say_hi()
    }
}

struct SuperOne;
struct SuperTwo;

impl Super for SuperOne {
    type T = BaseOne;
}
impl Super for SuperTwo {
    type T = BaseTwo;
}

trait AnySuper {
    fn say_hi(&self) -> &'static str;
}

impl<S> AnySuper for S where S : Super, S::T : Base {
    fn say_hi(&self) -> &'static str {
        Super::say_hi(self)
    }
}

fn main() {
    let one = Box::new(SuperOne);
    let two = Box::new(SuperTwo);
    
    let mut my_vec: Vec<Box<dyn AnySuper>> = Vec::new();
    
    my_vec.push(one);
    my_vec.push(two);
    
    println!("Success!");
}
```


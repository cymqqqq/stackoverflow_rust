Are Rust traits analogous to JavaScript mixins?

The [Rust book (2nd Edition)](https://doc.rust-lang.org/book/second-edition/ch10-02-traits.html) suggests that "Traits are similar to a feature often called ‘interfaces’ in other languages, though with some differences." For those not familiar with interfaces, the analogy doesn't illuminate. Can traits be reasonably thought of as [mixins](https://en.wikipedia.org/wiki/Mixin) such as those found commonly in JavaScript?

They both seem to be a way to share code and add methods to multiple types/objects without inheritance, but how crucial are the differences for conceptual understanding?

answer1

"Traits" (or "Roles" in Perl) are a way to add multiple units of functionality to a class (or struct in Rust) without the problems of multiple inheritance. Traits are "cross cutting concerns" meaning they're not part of the class hierarchy, they can be potentially implemented on any class.

Traits define an interface, meaning in order for anything to implement that trait it must define all the required methods. Like you can require that method parameters be of a certain classes, you can require that certain parameters implement certain traits.

A good example is writing output. In many languages, you have to decide if you're writing to a `FileHandle` object or a `Socket` object. This can get frustrating because sometimes things will only write to files, but not sockets or vice versa, or maybe you want to capture the output in a string for debugging.

If you instead define a trait, you can write to anything that implements that trait. This is exactly what Rust does with [`std::io::Write`](https://doc.rust-lang.org/std/io/trait.Write.html).

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;

    fn flush(&mut self) -> Result<()>;

    fn write_all(&mut self, mut buf: &[u8]) -> Result<()> {
        while !buf.is_empty() {
            match self.write(buf) {
                Ok(0) => return Err(Error::new(ErrorKind::WriteZero,
                                               "failed to write whole buffer")),
                Ok(n) => buf = &buf[n..],
                Err(ref e) if e.kind() == ErrorKind::Interrupted => {}
                Err(e) => return Err(e),
            }
        }
        Ok(())
    }

    ...and a few more...
}
```

Anything which wants to implement `Write` ***must\*** implement `write` and `flush`. A default `write_all` is provided, but you can implement your own if you like.

Here's how `Vec<u8>` implements `Write` so you can "print" to a vector of bytes.

```rust
impl Write for Vec<u8> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        self.extend_from_slice(buf);
        Ok(buf.len())
    }

    fn write_all(&mut self, buf: &[u8]) -> io::Result<()> {
        self.extend_from_slice(buf);
        Ok(())
    }

    fn flush(&mut self) -> io::Result<()> { Ok(()) }
}
```

Now when you write something that needs to output stuff instead of deciding if it should write to a [`File`](https://doc.rust-lang.org/std/fs/struct.File.html) or a [`TcpStream`](https://doc.rust-lang.org/std/net/struct.TcpStream.html) (a network socket) or whatever, you say it just has to have the `Write` trait.

```rust
fn display( out: Write ) {
    out.write(...whatever...)
}
```

------

Mixins are a severely watered down version of this. Mixins are a collection of methods which get injected into a class. That's about it. They solve the problem of multiple inheritance and cross-cutting concerns, but little else. There's no formal promise of an interface, you just call the methods and hope for the best.

Mixins are mostly functionally equivalent, but provide none of the compile time checks and high performance that traits do.

If you're familiar with mixins, traits will be a familiar way to compose functionality. The requirement to define an interface will be the struggle, but strong typing will be a struggle for anyone coming to Rust from JavaScript.

------

Unlike in JavaScript, where mixins are a neat add-on, traits are a fundamental part of Rust. They allow Rust to be strongly-typed, high-performance, very safe, but also extremely flexible. Traits allow Rust to perform extensive compile time checks on the validity of function arguments without the traditional restrictions of a strongly typed language.

Many core pieces of Rust are implemented with traits. `std::io::Writer` has already been mentioned. There's also `std::cmp::PartialEq` which handles `==` and `!=`. [`std::cmp::PartialOrd`](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html) for `>`, `>=`, `<` and `<=`. [`std::fmt::Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html) for how a thing should be printed with `{}`. And so on.

answer2

Traits or ["type classes"](https://en.wikipedia.org/wiki/Type_class) (in Haskell, [which is where Rust got traits from](https://air.mozilla.org/rust-typeclasses/)) are fundamentally about [logical](http://smallcultfollowing.com/babysteps/blog/2017/01/26/lowering-rust-traits-to-logic/) **constraints on types**. Traits are not fundamentally about values. Since JavaScript is unityped, mixins, which are about values, are nothing like traits/type-classes in a statically typed language like Rust or Haskell. **Traits let us talk in a principled way about the commonalities between types.** Unlike C++, which has "templates", Haskell and [Rust type check implementations **before monomorphization**](https://blog.rust-lang.org/2015/05/11/traits.html).

Assuming a generic function:

```rust
fn foo<T: Trait>(x: T) { /* .. */ }
```

or in Haskell:

```haskell
foo :: Trait t => t -> IO ()
foo = ...
```

The bound `T: Trait` means that any type `T` you pick must satisfy the `Trait`. To satisfy the `Trait`, the type must explicitly say that it is implementing the `Trait` and therein provide a definition of all items required by the `Trait`. In order to be sound, Rust also guarantees that each type implements a given trait at most once - therefore, there can never be overlapping implementations.

Consider the following marker trait and a type which implements it:

```rust
trait Foo {}
struct Bar;
impl Foo for Bar {}
```

or in Haskell:

```haskell
class Foo x where
data Bar = Bar
instance Foo Bar where
```

Notice that `Foo` does not have any methods, functions, or any other items. A difference between Haskell and Rust here is that `x` is absent in the Rust definition. This is because the first type parameter to a trait is implicit in Rust (and referred to by with `Self`) while it is explicit in Haskell.

Speaking of type parameters, we can define the trait `StudentOf` between two types like so:

```rust
trait StudentOf<A> {}
struct AlanTuring;
struct AlonzoChurch;
impl StudentOf<AlonzoChurch> for AlanTuring {}
```

or in Haskell:

```haskell
class StudentOf self a where
data AlanTuring = AlanTuring
data AlonzoChurch = AlonzoChurch
instance StudentOf AlanTuring AlonzoChurch where
```

Until now, we've not introduced any functions - let's do that:

```rust
trait From<T> {
    fn from(x: T) -> Self;
}

struct WrapF64(f64);
impl From<f64> for WrapF64 {
    fn from(x: f64) -> Self {
        WrapF64(x)
    }
}
```

or in Haskell:

```haskell
class From self t where
    from :: t -> self

newtype WrapDouble = WrapDouble Double
instance From WrapDouble Double where
    from d = WrapDouble d
```

What you've seen here is also a form of [return type polymorphism](https://eli.thegreenplace.net/2018/return-type-polymorphism-in-haskell/). Let's make it a bit more clear and consider a `Monoid` trait:

```rust
trait Monoid {
    fn mzero() -> Self;
    fn mappend(self, rhs: Self) -> Self;
}

struct Sum(usize);
impl Monoid for Sum {
    fn mzero() -> Self { Sum(0) }
    fn mappend(self, rhs: Self) -> Self { Sum(self.0 + rhs.0) }
}

fn main() {
    let s: Sum = Monoid::mzero();
    let s2 = s.mappend(Sum(2));
    // or equivalently:
    let s2 = <Sum as Monoid>::mappend(s, Sum(2));
}
```

or in Haskell:

```haskell
class Monoid m where
    mzero :: m    -- Notice that we don't have any inputs here.
    mappend :: m -> m -> m

...
```

The implementation of `mzero` here is inferred by the required return type `Sum`, which is why it is called return type polymorphism. Another subtle difference here is the `self` syntax in `mappend` - this is mostly a syntactic difference that allows us to do `s.mappend(Sum(2));` in Rust.

Traits also allow us to require that each type which implements the trait must provide an associated item, such as associated constants:

```rust
trait Identifiable {
    const ID: usize; // Each impl must provide a constant value.
}

impl Identifiable for bool {
    const ID: usize = 42;
}
```

or [associated types](https://doc.rust-lang.org/book/first-edition/associated-types.html):

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

struct Once<T>(Option<T>);
impl<T> Iterator for Once<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take()
    }
}
```

[Associated types](https://doc.rust-lang.org/book/first-edition/associated-types.html) also allow us to define functions on the type level rather than functions on the value level:

```rust
trait UnaryTypeFamily { type Output: Clone; }
impl UnaryTypeFamily for InputType { Output = String; }

fn main() {
    // Apply the function UnaryTypeFamily with InputType.
    let foo: <InputType as UnaryTypeFamily>::Output = String::new();
}
```

Some traits such as `Iterator` are also [object safe](https://stackoverflow.com/questions/44096235/understanding-traits-and-object-safety). This means that you can erase the actual type behind a pointer, and a vtable will be created for you:

```rust
fn use_boxed_iter(iter: Box<Iterator<Item = u8>>) { /* .. */ }
```

The Haskell equivalent of trait objects are [existentially quantified types](https://en.wikibooks.org/wiki/Haskell/Existentially_quantified_types#Example:_heterogeneous_lists), which in fact trait objects are in a type theoretical sense.

Finally, there's the issue of [higher kinded types](http://dev.stephendiehl.com/fun/001_basics.html#higher-kinded-types), which lets us be generic over type constructors. In Haskell, you can formulate what it means to be an (endo)functor like so:

```haskell
class Functor (f :: * -> *) where
    fmap :: (a -> b) -> (f a -> f b)
```

At this point, Rust does not have an equivalent notion, but will be equally expressive with [*generic associated types (GATs)*](http://smallcultfollowing.com/babysteps/blog/2016/11/02/associated-type-constructors-part-1-basic-concepts-and-introduction/) soon:

```rust
trait FunctorFamily {
    type Functor<T>;
    fn fmap<A, B, F>(self: Self::Functor<A>, mapper: F) -> Self::Functor<B>
    where F: Fn(A) -> B;
}
```
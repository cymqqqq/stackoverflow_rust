How can I create an iterator of &T from either a &Vec or Vec<&T>?

I have an enum with two variants. Either it contains a reference to a `Vec` of `String`s or it contains a `Vec` of references to `String`s:

```rust
enum Foo<'a> {
    Owned(&'a Vec<String>),
    Refs(Vec<&'a String>),
}
```

I want to iterate over references to the `String`s in this enum.

I tried to implement a method on `Foo`, but don't know how to make it return the right iterator:

```rust
impl<'a> Foo<'a> {
    fn get_items(&self) -> Iter<'a, String> {
        match self {
            Foo::Owned(v) => v.into_iter(),
            Foo::Refs(v) => /* what to put here? */,
        }
    }
}

fn main() {
    let test: Vec<String> = vec!["a".to_owned(), "b".to_owned()];
    let foo = Foo::Owned(&test);

    for item in foo.get_items() {
        // item should be of type &String here
        println!("{:?}", item);
    }
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=87aa534962775da8816495734901607b)

What is an idiomatic method to achieve this abstraction over `&Vec<T>` and `Vec<&T>`? `get_items` may also return something different, as long as it implements the `IntoIterator` trait so that I can use it in the `for` loop.

answer1

You can't just use the `std::slice::Iter` type for this.

If you don't want to copy the strings or vector, you'll have to implement your own iterator, for example:

```rust
struct FooIter<'a, 'b> {
    idx: usize,
    foo: &'b Foo<'a>,
}

impl<'a, 'b> Iterator for FooIter<'a, 'b> {
    type Item = &'a String;
    fn next(&mut self) -> Option<Self::Item> {
        self.idx += 1;
        match self.foo {
            Foo::Owned(v) => v.get(self.idx - 1),
            Foo::Refs(v) => v.get(self.idx - 1).map(|s| *s),
        }
    }
}

impl<'a, 'b> Foo<'a> {
    fn get_items(&'b self) -> FooIter<'a, 'b> {
        FooIter { idx: 0, foo: self }
    }
}

fn main() {
    let test: Vec<String> = vec!["a".to_owned(), "b".to_owned()];
    let foo = Foo::Owned(&test);
    for item in foo.get_items() {
        println!("{:?}", item);
    }
    let a = "a".to_string();
    let b = "b".to_string();
    let test: Vec<&String> = vec![&a, &b];
    let foo = Foo::Refs(test);
    for item in foo.get_items() {
        println!("{:?}", item);
    }
}
```

answer2

There is a handy crate, [`auto_enums`](https://crates.io/crates/auto_enums), which can generate a type for you so a function can have multiple return types, as long as they implement the same trait. It's similar to the code in [Denys Séguret's answer](https://stackoverflow.com/a/58050166/493729) except it's all done for you by the `auto_enum` macro:

```rust
use auto_enums::auto_enum;

impl<'a> Foo<'a> {
    #[auto_enum(Iterator)]
    fn get_items(&self) -> impl Iterator<Item = &String> {
        match self {
            Foo::Owned(v) => v.iter(),
            Foo::Refs(v) => v.iter().copied(),
        }
    }
}
```

Add the dependency by adding this in your `Cargo.toml`:

```rust
[dependencies]
auto_enums = "0.6.3"
```

answer3

If you don't want to implement your own iterator, you need dynamic dispatch for this, because you want to return different iterators depending on the enum variant.

We need a **trait object** (`&dyn Trait`, `&mut dyn Trait` or `Box<dyn Trait>`) to use dynamic dispatch:

```rust
impl<'a> Foo<'a> {
    fn get_items(&'a self) -> Box<dyn Iterator<Item = &String> + 'a> {
        match self {
            Foo::Owned(v) => Box::new(v.into_iter()),
            Foo::Refs(v) => Box::new(v.iter().copied()),
        }
    }
}
```

[`.copied()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.copied) converts the `Iterator<Item = &&String>` into an `Iterator<Item = &String>`, so this doesn't actually copy anything :)

answer4

A few things you should know first:

- You are most definitely going to have two different iterators, because they're different base types you're iterating over. Therefore I'm going to use a `Box<dyn Iterator<Item = &'a _>>`, but feel free to use an `enum` if this causes a quantifiable performance drop.
- You need to introduce `self`'s lifetime here, because what if we return an iterator whose lifetime is `'a`, but `'a > 'self`? Therefore we make a new lifetime (Which I'll call `'b`.).
- Now it's just a matter of wrangling with the reference layers:

Here's the implementation using the original types:

```rust
enum Foo<'a> {
    Owned(&'a Vec<String>),
    Refs(Vec<&'a String>)
}

impl<'a> Foo<'a> {
    fn get_items<'b>(&'b self) -> Box<dyn Iterator<Item = &'a String> + 'b> {
        match self {
            Foo::Owned(v) => //v: &'a Vec<String>
                Box::new(
                    v.iter() //Iterator<Item = &'a String> -- Good!
                ),
            Foo::Refs(v) => //v: Vec<&'a String>
                Box::new(
                    v.iter() //Iterator<Item = &'b &'a String> -- Bad!
                        .map(|x| *x) //Iterator<Item = &'a String> -- Good!
                ),
        }
    }
}
```

These types aren't really rust-like (Or more formally, *idiomatic*), so here's that version using slices and `str`s:

```rust
enum Foo<'a> {
    Owned(&'a [String]),
    Refs(Vec<&'a str>)
}

impl<'a> Foo<'a> {
    fn get_items<'b>(&'b self) -> Box<dyn Iterator<Item = &'a str> + 'b> {
        match self {
            Foo::Owned(v) => 
                Box::new(
                    v.into_iter()
                        .map(|x| &**x) //&'a String -> &'a str
                ),
            Foo::Refs(v) =>
                Box::new(
                    v.iter()
                        .map(|x| *x) //&'b &'a str -> &'a str
                )/* what to put here? */,
        }
    }
}
```



answer5

Ideally you would want:

```rust
fn get_items(&self) -> impl Iterator<Item = &String> {
    match self {
        Foo::Owned(v) => v.into_iter(),
        Foo::Refs(v)  => v.iter().copied(),
    }
}
```

The call to `copied` is here to convert an `Iterator<Item = &&String>` into the `Iterator<Item = &String>` we want. This doesn't work because the two match arms have different types:

```none
error[E0308]: match arms have incompatible types
  --> src/main.rs:12:30
   |
10 | /         match self {
11 | |             Foo::Owned(v) => v.into_iter(),
   | |                              ------------- this is found to be of type `std::slice::Iter<'_, std::string::String>`
12 | |             Foo::Refs(v)  => v.iter().copied(),
   | |                              ^^^^^^^^^^^^^^^^^ expected struct `std::slice::Iter`, found struct `std::iter::Copied`
13 | |         }
   | |_________- `match` arms have incompatible types
   |
   = note: expected type `std::slice::Iter<'_, std::string::String>`
              found type `std::iter::Copied<std::slice::Iter<'_, &std::string::String>>`
```

You can fix this error thanks to the [`itertools`](https://crates.io/crates/itertools) or [`either`](https://crates.io/crates/either) crates, which contain a handy adapter called [`Either`](https://docs.rs/itertools/0.8.0/itertools/enum.Either.html) ([*](https://docs.rs/either/1.5.3/either/enum.Either.html)) that allows you to choose dynamically between two iterators:

```rust
fn get_items(&self) -> impl Iterator<Item = &String> {
    match self {
        Foo::Owned(v) => Either::Left(v.into_iter()),
        Foo::Refs(v)  => Either::Right(v.iter().copied()),
    }
}
```

[playground](https://play.integer32.com/?version=stable&mode=debug&edition=2018&gist=da56db4c922bf335c209f8ada9b8a66b)


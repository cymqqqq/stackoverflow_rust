Trait to store structs with different generic parameters

I need to store in the same `Vec` instances of the same struct, but with different generic parameters. This is the struct definition:

```rust
struct Struct<'a, T: 'a> {
    items: Vec<&'a T>
}
```

The struct has a method returning an iterator to a type that does not depend on the generic type parameter `T`:

```rust
impl<'a, T: 'a> Struct<'a, T> {
    fn iter(&self) -> slice::Iter<&i32> {
        unimplemented!()
    }
}
```

I need to access this method for those different structs in the vector, so I've implemented this trait:

```rust
type Iter<'a> = Iterator<Item=&'a i32>;

trait Trait {
    fn iter(&self) -> Box<Iter>;
}
```

And I've implemented the trait for `Struct`:

```rust
impl<'a, T: 'a> Trait for Struct<'a, T> {
    fn iter(&self) -> Box<Iter> {
        Box::new(self.iter())
    }
}
```

But the compiler complains:

```rust
<anon>:21:9: 21:30 error: type mismatch resolving `<core::slice::Iter<'_, &i32> as core::iter::Iterator>::Item == &i32`:
expected &-ptr,
    found i32 [E0271]
<anon>:21         Box::new(self.iter())
                  ^~~~~~~~~~~~~~~~~~~~~
<anon>:21:9: 21:30 help: see the detailed explanation for E0271
<anon>:21:9: 21:30 note: required for the cast to the object type `core::iter::Iterator<Item=&i32> + 'static`
<anon>:21         Box::new(self.iter())
                  ^~~~~~~~~~~~~~~~~~~~~
```

I've tried different possibilities for lifetime parameters in the trait, but none of them work. How can I make this work?

[Rust Playground snippet](http://is.gd/FDPzKY)

**Edit**

As pointed out by @MatthieuM. one problem is that the type alias is not working properly. Here's another example demonstrating this:

```rust
use std::slice;

type Iter<'a> = Iterator<Item=&'a i32>;

struct Struct<'a> { _phantom: std::marker::PhantomData<&'a i32> }

impl<'a> Struct<'a> {
    fn direct<'b>(i: &'b slice::Iter<'a, i32>) -> &'b Iterator<Item=&'a i32>
    { i }

    fn aliased<'b>(i: &'b slice::Iter<'a, i32>) -> &'b Iter<'a>
    { i }
}
```

In this example, `direct` compiles, but `aliased` not, with the error:

```rust
<anon>:12:7: 12:8 error: the type `core::slice::Iter<'a, i32>` does not fulfill the required lifetime
<anon>:12     { i }
                ^
note: type must outlive the static lifetime
```

But they seem to be the same thing. What's happening?

answer1

Problem 1 — `slice::Iter<T>` has an `Iterator::Item` of `&T`, thus your reference levels are mismatched. Change your method to be

```rust
fn iter(&self) -> slice::Iter<i32>
```

Problem 2 — `Box<SomeTrait>` is equivalent to `Box<SomeTrait + 'static>`, but your iterator does not live for the `'static` lifetime. You need to explicitly bring in a lifetime:

```rust
Box<SomeTrait + 'a>
```

Problem 3 — I don't understand how you can create a type alias for a trait, that seems very odd. You probably don't want it anyway. Instead, create a type alias for the whole boxed version:

```rust
type IterBox<'a> = Box<Iterator<Item=&'a i32> + 'a>;
```

Problem 4 — Rearrange your `main` so that references will live long enough and add mutability:

```rust
fn main() {
    let i = 3;
    let v = vec![&i];
    let mut traits : Vec<Box<Trait>> = Vec::new();
    traits.push(Box::new(Struct{ items: v }));
}
```

All together:

```rust
use std::slice;

type IterBox<'a> = Box<Iterator<Item=&'a i32> + 'a>;

trait Trait {
    fn iter<'a>(&'a self) -> IterBox;
}

struct Struct<'a, T: 'a> {
    items: Vec<&'a T>
}

impl<'a, T: 'a> Struct<'a, T> {
    fn iter(&self) -> slice::Iter<i32> {
        unimplemented!()
    }
}

impl<'a, T: 'a> Trait for Struct<'a, T> {
    fn iter(&self) -> IterBox {
        Box::new(self.iter())
    }
}

fn main() {
    let i = 3;
    let v = vec![&i];
    let mut traits: Vec<Box<Trait>> = Vec::new();
    traits.push(Box::new(Struct { items: v }));
}
```


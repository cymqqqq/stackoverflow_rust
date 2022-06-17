Struct with a generic trait which is also a generic trait

In Rust 1.15, I have created a trait to abstract over reading & parsing file format(s). I'm trying to create a struct which has this generic trait inside.

I have this trait:

```rust
use std::io::Read;

trait MyReader<R: Read> {
    fn new(R) -> Self;
    fn into_inner(self) -> R;

    fn get_next(&mut self) -> Option<u32>;
    fn do_thingie(&mut self);
}
```

I want to make a struct which has a reference to something that implements this.

```rust
struct MyIterThing<'a, T: MyReader<R>+'a> {
    inner: &'a mut T,
}
```

Gives the following error:

```none
error[E0412]: type name `R` is undefined or not in scope
  --> <anon>:11:36
   |
11 | struct MyIterThing<'a, T: MyReader<R>+'a> {
   |                                    ^ undefined or not in scope
   |
   = help: no candidates by the name of `R` found in your project; maybe you misspelled the name or forgot to import an external crate?
```

`T: MyReader+'a`, I get the error: `"error[E0243]: wrong number of type arguments: expected 1, found 0"`, `T: MyReader<R: Read>+'a` gives a low level syntax error, it's not expecting a `:` there.

And this doesn't work either:

```none
error[E0392]: parameter `R` is never used
  --> <anon>:11:24
   |
11 | struct MyIterThing<'a, R: Read, T: MyReader<R>+'a> {
   |                        ^ unused type parameter
   |
   = help: consider removing `R` or using a marker such as `std::marker::PhantomData`
```

How do I create my `MyIterThing` struct?

answer1

You probably don't want a type parameter, you want an *associated type*:

```rust
use std::io::Read;

trait MyReader {
    type R: Read;

    fn new(Self::R) -> Self;
    fn into_inner(self) -> Self::R;

    fn get_next(&mut self) -> Option<u32>;
    fn do_thingie(&mut self);
}

struct MyIterThing<'a, T>
    where T: MyReader + 'a
{
    inner: &'a mut T,
}

fn main() {}
```

answer2

The error message gives you a suggestion to use a marker, like [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html). You can do it like this:

```rust
use std::marker::PhantomData;

struct MyIterThing<'a, R: Read, T: MyReader<R> + 'a> {
    inner: &'a mut T,
    marker: PhantomData<R>,
}
```

Instances of `PhantomData` have zero runtime cost, so it's better to use that than to just create a field of type `R`.

------

Another solution would be to use an associated type instead of a type parameter:

```rust
trait MyReader {
    type Source: Read;

    fn new(Self::Source) -> Self;
    fn into_inner(self) -> Self::Source;

    fn get_next(&mut self) -> Option<u32>;
    fn do_thingie(&mut self);
}

struct MyIterThing<'a, T: MyReader + 'a> {
    inner: &'a mut T,
}
```

This is a little less flexible as there can only be one choice of `Source` per implementation of `MyReader`, but it could be sufficient, depending on your needs.


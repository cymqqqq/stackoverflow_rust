How can I create my own data structure with an iterator that returns mutable references?

I have created a data structure in Rust and I want to create iterators for it. Immutable iterators are easy enough. I currently have this, and it works fine:

```rust
// This is a mock of the "real" EdgeIndexes class as
// the one in my real program is somewhat complex, but
// of identical type

struct EdgeIndexes;

impl Iterator for EdgeIndexes {
    type Item = usize;
    fn next(&mut self) -> Option<Self::Item> {
        Some(0)
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}

pub struct CGraph<E> {
    nodes: usize,
    edges: Vec<E>,
}

pub struct Edges<'a, E: 'a> {
    index: EdgeIndexes,
    graph: &'a CGraph<E>,
}

impl<'a, E> Iterator for Edges<'a, E> {
    type Item = &'a E;

    fn next(&mut self) -> Option<Self::Item> {
        match self.index.next() {
            None => None,
            Some(x) => Some(&self.graph.edges[x]),
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        self.index.size_hint()
    }
}
```

I want to create a iterator that returns mutable references as well. I've tried doing this, but can't find a way to get it to compile:

```rust
pub struct MutEdges<'a, E: 'a> {
    index: EdgeIndexes,
    graph: &'a mut CGraph<E>,
}

impl<'a, E> Iterator for MutEdges<'a, E> {
    type Item = &'a mut E;

    fn next(&mut self) -> Option<&'a mut E> {
        match self.index.next() {
            None => None,
            Some(x) => self.graph.edges.get_mut(x),
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        self.index.size_hint()
    }
}
```

Compiling this results in the following error:

```none
error[E0495]: cannot infer an appropriate lifetime for lifetime parameter in function call due to conflicting requirements
  --> src/lib.rs:54:24
   |
54 |             Some(x) => self.graph.edges.get_mut(x),
   |                        ^^^^^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 51:5...
  --> src/lib.rs:51:5
   |
51 | /     fn next(&mut self) -> Option<&'a mut E> {
52 | |         match self.index.next() {
53 | |             None => None,
54 | |             Some(x) => self.graph.edges.get_mut(x),
55 | |         }
56 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/lib.rs:54:24
   |
54 |             Some(x) => self.graph.edges.get_mut(x),
   |                        ^^^^^^^^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 48:6...
  --> src/lib.rs:48:6
   |
48 | impl<'a, E> Iterator for MutEdges<'a, E> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut E>
              found std::option::Option<&mut E>
```

I'm unsure how to interpret these errors and how to change my code in order to allow `MutEdges` to return mutable references.

Link to [playground with code](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e36e3baeffce5c7c19a1921b02a336dc).

answer1

You can't compile this because mutable references are more restrictive than immutable references. A shortened version that illustrates the issue is this:

```rust
struct MutIntRef<'a> {
    r: &'a mut i32
}

impl<'a> MutIntRef<'a> {
    fn mut_get(&mut self) -> &'a mut i32 {
        &mut *self.r
    }
}

fn main() {
    let mut i = 42;
    let mut mir = MutIntRef { r: &mut i };
    let p = mir.mut_get();
    let q = mir.mut_get();
    println!("{}, {}", p, q);
}
```

Which produces the same error:

```none
error[E0495]: cannot infer an appropriate lifetime for borrow expression due to conflicting requirements
 --> src/main.rs:7:9
  |
7 |         &mut *self.r
  |         ^^^^^^^^^^^^
  |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 6:5...
 --> src/main.rs:6:5
  |
6 | /     fn mut_get(&mut self) -> &'a mut i32 {
7 | |         &mut *self.r
8 | |     }
  | |_____^
note: ...so that reference does not outlive borrowed content
 --> src/main.rs:7:9
  |
7 |         &mut *self.r
  |         ^^^^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 5:6...
 --> src/main.rs:5:6
  |
5 | impl<'a> MutIntRef<'a> {
  |      ^^
note: ...so that reference does not outlive borrowed content
 --> src/main.rs:7:9
  |
7 |         &mut *self.r
  |         ^^^^^^^^^^^^
```

If we take a look at the main function, we get two mutable references called `p` and `q` that both alias the memory location of `i`. This is not allowed. In Rust, we can't have two mutable references that alias and are both usable. The motivation for this restriction is the observation that mutation and aliasing don't play well together with respect to memory safety. So, it's good that the compiler rejected the code. If something like this compiled, it would be easy to get all kinds of memory corruption errors.

The way Rust avoids this kind of danger is by keeping at most *one* mutable reference usable. So, if you want to create mutable reference to X based on a mutable reference to Y where X is owned by Y, we better make sure that as long as the reference to X exists, we can't touch the other reference to Y anymore. In Rust this is achieved through lifetimes and borrowing. The compiler considers the original reference to be borrowed in such a case and this has an effect on the lifetime parameter of the resulting reference as well. If we change

```rust
fn mut_get(&mut self) -> &'a mut i32 {
    &mut *self.r
}
```

to

```rust
fn mut_get(&mut self) -> &mut i32 { // <-- no 'a anymore
    &mut *self.r // Ok!
}
```

the compiler stops complaining about this `get_mut` function. It now returns a reference with a lifetime parameter that matches `&self` and not `'a` anymore. This makes `mut_get` a function with which you "borrow" `self`. And that's why the compiler complains about a different location:

```none
error[E0499]: cannot borrow `mir` as mutable more than once at a time
  --> src/main.rs:15:13
   |
14 |     let p = mir.mut_get();
   |             --- first mutable borrow occurs here
15 |     let q = mir.mut_get();
   |             ^^^ second mutable borrow occurs here
16 |     println!("{}, {}", p, q);
   |                        - first borrow later used here
```

Apparently, the compiler really *did* consider `mir` to be borrowed. This is good. This means that now there is only one way of reaching the memory location of `i`: `p`.

Now you may wonder: How did the standard library authors manage to write the mutable vector iterator? The answer is simple: They used unsafe code. There is no other way. The Rust compiler simply does not know that whenever you ask a mutable vector iterator for the next element, that you get a different reference every time and never the same reference twice. Of course, we know that such an iterator won't give you the same reference twice and that makes it safe to offer this kind of interface you are used to. We don't need to "freeze" such an iterator. If the references an iterator returns don't overlap, it's safe to *not* have to borrow the iterator for accessing an element. Internally, this is done using unsafe code (turning raw pointers into references).

The easy **solution** for your problem might be to rely on `MutItems`. This is already a library-provided mutable iterator over a vector. So, you might get away with just using that instead of your own custom type, or you could wrap it inside your custom iterator type. In case you can't do that for some reason, you would have to write your own unsafe code. And if you do so, make sure that

- You don't create multiple mutable references that alias. If you did, this would violate the Rust rules and invoke undefined behavior.
- You don't forget to use the `PhantomData` type to tell the compiler that your iterator is a reference-like type where replacing the lifetime with a longer one is not allowed and could otherwise create a dangling iterator.
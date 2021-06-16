In Rust, how can a reference be a pointer to a pointer-to-a-pointer?

Today's Rust mystery is from section 4.9 of The Rust Programming Language, First Edition. The example of references and borrowing has this example:

```rust
fn main() {
    fn sum_vec(v: &Vec<i32>) -> i32 {
        return v.iter().fold(0, |a, &b| a + b);
    }

    fn foo(v1: &Vec<i32>) -> i32 {
        sum_vec(v1);
    }

    let v1 = vec![1, 2, 3];

    let answer = foo(&v1);
    println!("{}", answer);
}
```

That seems reasonable. It prints "6", which is what you'd expect if the `v` of `sum_vec` is a C++ reference; it's just a name for a memory location, the vector `v1` we defined in `main()`.

Then I replaced the body of `sum_vec` with this:

```rust
fn sum_vec(v: &Vec<i32>) -> i32 {
    return (*v).iter().fold(0, |a, &b| a + b);
}
```

It compiled and worked as expected. Okay, that's not… entirely crazy. The compiler is trying to make my life easier, I get that. Confusing, something that I have to memorize as a specific tic of the language, but not entirely crazy. Then I tried:

```rust
fn sum_vec(v: &Vec<i32>) -> i32 {
    return (**v).iter().fold(0, |a, &b| a + b);
}
```

It still worked! What the hell?

```rust
fn sum_vec(v: &Vec<i32>) -> i32 {
    return (***v).iter().fold(0, |a, &b| a + b);
}
```

`type [i32] cannot be dereferenced`. Oh, thank god, something that makes sense. But I would have expected that almost two iterations earlier!

References in Rust aren't C++ "names for another place in memory," but what *are* they? They're not pointers either, and the rules about them seem to be either esoteric or highly ad-hoc. What is happening such that a reference, a pointer, and a pointer-to-a-pointer all work equally well here?

answer

The rules are not *ad-hoc* nor really esoteric. Inspect the type of `v` and it's various dereferences:

```rust
fn sum_vec(v: &Vec<i32>) {
    let () = v;
}
```

You'll get:

1. `v` -> `&std::vec::Vec<i32>`
2. `*v` -> `std::vec::Vec<i32>`
3. `**v` -> `[i32]`

The first dereference you already understand. The second dereference is thanks to the [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html) trait. `Vec<T>` dereferences to `[T]`.

When performing method lookup, [there's a straight-forward set of rules](https://stackoverflow.com/q/28519997/155423):

1. If the type has the method, use it and exit the lookup.
2. If a reference to the type has the method, use it and exit the lookup.
3. If the type can be dereferenced, do so, then return to step 1.
4. Else the lookup fails.

> References in Rust aren't C++ "names for another place in memory,"

They absolutely are names for a place in memory. In fact, they compile down to the same C / C++ pointer you know.

notes

When you actually work out the method call by hand, it looks a bit like `let v = &vec![42]; <[_]>::iter(&**v);`. The sequence is: Reference to Vec, Vec, Slice, Reference to Slice
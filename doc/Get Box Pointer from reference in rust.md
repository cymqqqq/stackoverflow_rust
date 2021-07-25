Get Box Pointer from reference in rust

Say I have a function like this:

```rust
fn foo(item: &FooType) -> Box<FooType>{
    Box::new(item)
}
```

I get the following error:

> mismatched types: expected enum `FooType`, found `&FooType`

I'm assuming I need to do some reference/boxing and lifetime magic, but as a rust beginner, I'm not 100% sure how to get this working.

answer1

I think there is no definitive answer since it depends on what you expect.

If you want to obtain a heap-allocated clone of something you want to keep usable, then cloning in `foo1()` can help.

A better solution could be to pass a **value** not a reference, as in `foo2()`, because this way the call site can decide if a clone is necessary or if moving the original value into the box is enough.

You may also want the box to hold the reference instead of the value, as in `foo3()`, but this is like a code-smell to me (many indirections that make reasoning about lifetimes very complicated).

```rust
#[derive(Debug, Clone)]
struct FooType {
    value: usize,
}

fn foo1(item: &FooType) -> Box<FooType> {
    Box::new(item.clone())
}

fn foo2(item: FooType) -> Box<FooType> {
    Box::new(item)
}

fn foo3(item: &FooType) -> Box<&FooType> {
    Box::new(item)
}

fn main() {
    let ft_a = FooType { value: 12 };
    let bft_a = foo1(&ft_a);
    println!("bft_a: {:?}", bft_a);
    println!("ft_a: {:?}", ft_a);
    println!("~~~~~~~~~~~~~~~~~~");
    let ft_b = FooType { value: 34 };
    let bft_b = foo2(ft_b);
    println!("bft_b: {:?}", bft_b);
    //  println!("ft_b: {:?}", ft_b); // borrow of moved value: `ft_b`
    println!("ft_b is consumed");
    println!("~~~~~~~~~~~~~~~~~~");
    let ft_c = FooType { value: 56 };
    let bft_c = foo2(ft_c.clone());
    println!("bft_c: {:?}", bft_c);
    println!("ft_c: {:?}", ft_c);
    println!("~~~~~~~~~~~~~~~~~~");
    let ft_d = FooType { value: 78 };
  	let bft_d = foo3(&ft_d);
    println!("bft_d: {:?}", bft_d);
    println!("ft_d: {:?}", ft_d);
}
/*
bft_a: FooType { value: 12 }
ft_a: FooType { value: 12 }
~~~~~~~~~~~~~~~~~~
bft_b: FooType { value: 34 }
ft_b is consumed
~~~~~~~~~~~~~~~~~~
bft_c: FooType { value: 56 }
ft_c: FooType { value: 56 }
~~~~~~~~~~~~~~~~~~
bft_d: FooType { value: 78 }
ft_d: FooType { value: 78 }
*/
```

answer2

The compiler is complaining because in the function's signature you specified `Box<FooType>` as the return type, and you are giving it a `Box<&FooType>` when you wrap the `ìtem` in a `Box`.

Change the function's signature so that it receives a `FooType` rather than a reference to it like this:

```rust
fn foo(item: FooType) -> Box<FooType>{
    Box::new(item)
}
```


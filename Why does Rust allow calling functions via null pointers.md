Why does Rust allow calling functions via null pointers?

I was experimenting with function pointer magic in Rust and ended up with a code snippet which I have absolutely no explanation for why it compiles and even more, why it runs.

```rust
fn foo() {
    println!("This is really weird...");
}

fn caller<F>() where F: FnMut() {
    let closure_ptr = 0 as *mut F;
    let closure = unsafe { &mut *closure_ptr };
    closure();
}

fn create<F>(_: F) where F: FnMut() {
    caller::<F>();
}

fn main() {
    create(foo);
    
    create(|| println!("Okay..."));
    
    let val = 42;
    create(|| println!("This will seg fault: {}", val));
}
```

I cannot explain *why* `foo` is being invoked by casting a null pointer in `caller(...)` to an instance of type `F`. I would have thought that functions may only be called through corresponding function pointers, but that clearly can't be the case given that the pointer itself is null. With that being said, it seems that I clearly misunderstand an important piece of Rust's type system.

[Example on Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e80717402ba5930d350e140b7bacf282)

answer1

This program never actually constructs a function pointer at all- it always invokes `foo` and those two closures *directly.*

Every Rust function, whether it's a closure or a `fn` item, has a unique, anonymous type. This type implements the `Fn`/`FnMut`/`FnOnce` traits, as appropriate. The anonymous type of a `fn` item is zero-sized, just like the type of a closure with no captures.

Thus, the expression `create(foo)` instantiates `create`'s parameter `F` with `foo`'s type- this is not the function pointer type `fn()`, but an anonymous, zero-sized type just for `foo`. In error messages, rustc calls this type `fn() {foo}`, as you can see [this error message](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=2c30794e639b53cb5f59212d1fabd174).

Inside `create::<fn() {foo}>` (using the name from the error message), the expression `caller::<F>()` forwards this type to `caller` without giving it a value of that type.

Finally, in `caller::<fn() {foo}>` the expression `closure()` desugars to `FnMut::call_mut(closure)`. Because `closure` has type `&mut F` where `F` is just the zero-sized type `fn() {foo}`, the `0` value of `closure` itself is simply never used1, and the program calls `foo` directly.

The same logic applies to the closure `|| println!("Okay...")`, which like `foo` has an anonymous zero-sized type, this time called something like [`[closure@src/main.rs:2:14: 2:36\]`](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=7d4f09ea0eb0cb82c57c3e3cb0519c29).

The second closure is not so lucky- its type is *not* zero-sized, because it must contain a reference to the variable `val`. This time, `FnMut::call_mut(closure)` actually needs to dereference `closure` to do its job. So it crashes2.

------

1 Constructing a null reference like this is technically undefined behavior, so the compiler makes no promises about this program's overall behavior. However, replacing `0` with some other "address" with the alignment of `F` avoids that problem for zero-sized types like `fn() {foo}`, and gives [the same behavior](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=847ca808ae5f44e2315fabdf05432352)!)

2 Again, constructing a null (or dangling) reference is the operation that actually takes the blame here- after that, anything goes. A segfault is just one possibility- a future version of rustc, or the same version when run on a slightly different program, might do something else entirely!

notes:

 If you actually want to materialize a dangling reference to zero-sized `F`, [`NonNull::::dangling()`](https://doc.rust-lang.org/std/ptr/struct.NonNull.html#method.dangling) is a better way than `align_of::<F>() as *mut F`. It does the same thing but with less chance of making a mistake that leads to unsoundness. (For illustration purposes, `align_of` works better.) 

answer2

The [type of `fn foo() {...}`](https://doc.rust-lang.org/reference/types/function-item.html) is not a function pointer `fn()`, it's actually a unique type specific to `foo`. As long as you carry that type along (here as `F`), the compiler knows how to call it without needing any extra pointers (a value of such a type carries no data). A closure that doesn't capture anything works the same way. It only gets dicey when the last closure tries to look up `val` because you put a `0` where (presumably) the pointer to `val` was supposed to be.

You can observe this with `size_of`, in the first two calls, the size of `closure` is zero, but in the last call with something captured in the closure, the size is 8 (at least on the playground). If the size is 0, the program doesn't have to load anything from the `NULL` pointer.

The effective cast of a `NULL` pointer to a reference is still undefined behavior, but because of type shenanigans and not because of memory access shenanigans: having references that are really `NULL` is in itself illegal, because memory layout of types like `Option<&T>` relies on the assumption that the value of a reference is never `NULL`. Here's an example of how it can go wrong:

```rust
unsafe fn null<T>(_: T) -> &'static mut T {
    &mut *(0 as *mut T)
}

fn foo() {
    println!("Hello, world!");
}

fn main() {
    unsafe {
        let x = null(foo);
        x(); // prints "Hello, world!"
        let y = Some(x);
        println!("{:?}", y.is_some()); // prints "false", y is None!
    }
}
```

answer3

Although this is entirely up to UB, here's what I assume might be happening in the two cases:

1. The type `F` is a closure with no data. This is equivalent to a function, which means that `F` is a function item. What this means is that the compiler can optimize any call to an `F` into a call to whatever function produced `F` (without ever making a function pointer). See [this](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=eac19b8b9c82f734fab2cd639d737468) for an example of the different names for these things.
2. The compiler sees that `val` is always 42, and hence it can optimize it into a constant. If that's the case, then the closure passed into `create` is again a closure with no captured items, and hence we can follow the ideas in #1.

Additionally, I say this is UB, however please note something critical about UB: If you invoke UB and the compiler takes advantage of it in an unexpected way, it is not *trying* to mess you up, it is *trying* to optimize your code. UB after all, is about the compiler mis-optimizing things because you've broken some expectations it has. It is hence, completely logical that the compiler optimizes this way. It would also be completely logical that the compiler doesn't optimize this way and instead takes advantage of the UB.

extended answer

This is "working" because `fn() {foo}` and the first closure are zero-sized types. Extended answer:

If this program ends up executed in Miri (Undefined behaviour checker), it ends up failing because NULL pointer is dereferenced. NULL pointer cannot ever be dereferenced, even for zero-sized types. However, undefined behaviour can do anything, so compiler makes no promises about the behavior, and this means it can break in the future release of Rust.

```rust
error: Undefined Behavior: memory access failed: 0x0 is not a valid pointer
  --> src/main.rs:7:28
   |
7  |     let closure = unsafe { &mut *closure_ptr };
   |                            ^^^^^^^^^^^^^^^^^ memory access failed: 0x0 is not a valid pointer
   |
   = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
   = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
           
   = note: inside `caller::<fn() {foo}>` at src/main.rs:7:28
note: inside `create::<fn() {foo}>` at src/main.rs:13:5
  --> src/main.rs:13:5
   |
13 |     func_ptr();
   |     ^^^^^^^^^^
note: inside `main` at src/main.rs:17:5
  --> src/main.rs:17:5
   |
17 |     create(foo);
   |     ^^^^^^^^^^^
```

This issue can be easily fixed by writing `let closure_ptr = 1 as *mut F;`, then it will only fail on line 22 with the second closure that will segfault.

```rust
error: Undefined Behavior: inbounds test failed: 0x1 is not a valid pointer
  --> src/main.rs:7:28
   |
7  |     let closure = unsafe { &mut *closure_ptr };
   |                            ^^^^^^^^^^^^^^^^^ inbounds test failed: 0x1 is not a valid pointer
   |
   = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
   = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
           
   = note: inside `caller::<[closure@src/main.rs:22:12: 22:55 val:&i32]>` at src/main.rs:7:28
note: inside `create::<[closure@src/main.rs:22:12: 22:55 val:&i32]>` at src/main.rs:13:5
  --> src/main.rs:13:5
   |
13 |     func_ptr();
   |     ^^^^^^^^^^
note: inside `main` at src/main.rs:22:5
  --> src/main.rs:22:5
   |
22 |     create(|| println!("This will seg fault: {}", val));
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Why it didn't complain about `foo` or `|| println!("Okay...")`? Well, because they don't store any data. When referring to a function, you don't get a function pointer but rather a zero-sized type representing that specific function - this helps with monomorphization, as each function is distinct. A structure not storing any data can be created from aligned dangling pointer.

However, if you explicitly say the function is a function pointer by saying `create::<fn()>(foo)` then the program will stop working.


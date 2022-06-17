Understand smart pointers in Rust

I am a newbie to Rust and writing to understand the "Smart pointers" in Rust. I have basic understanding of how smart pointers works in C++ and has been using it for memory management since a few years ago. But to my very much surprise, Rust also provides such utility **explicitly**.

Because from a tutorial here (https://pcwalton.github.io/2013/03/18/an-overview-of-memory-management-in-rust.html), it seems that every raw pointers have been automatically wrapped with a smart pointer, which seems very reasonable. Then why do we still need such `Box<T>`, `Rc<T>`, and `Ref<T>` stuff? According to this specification: https://doc.rust-lang.org/book/ch15-00-smart-pointers.html

Any comments will be apprecicated a lot. Thanks.

answer

You can think about the difference between a `T` and a `Box<T>` as the difference between a statically allocated object and a dynamically allocated object (the latter being created via a `new` expression in C++ terms).

In Rust, both `T` and `Box<T>` represent a variable that has *ownership* over the referent object (i.e. when the variable goes out of scope, the object will be destroyed, whether it was stored by value or by reference). On the contrary, `&T` and `&mut T` represent *borrowing* of the object (i.e. these variables are not responsible for destroying the object, and they cannot outlive the owner of the object).

By default, you'd probably want to use `T`, but sometimes you might want (or have) to use `Box<T>`. For example, you would use a `Box<T>` if you want to own a `T` that's too large to be allocated in place. You would also use it when the object doesn't have a known size at all, which means that your only choice to store it or pass it around is through the "pointer" (the `Box<T>`).

------

In Rust, an object is generally either mutable or aliased, but not both. If you have given out immutable references to an object, you normally need to wait until those references are over before you can mutate that object again.

Additionally, Rust's immutability is transitive. If you receive an object immutably, it means that you have access to its contents (and the contents of those contents, and so on) also immutably.

Normally, all of these things are enforced at compile time. This means that you catch errors faster, but you are limited to being able to express only what the compiler can prove statically.

Like `T` and `Box<T>`, you may sometimes use `RefCell<T>`, which is another ownership type. But unlike `T` and `Box<T>`, the `RefCell<T>` enforces the borrow checking rules *at runtime* instead of compile time, meaning that sometimes you can do things with it that are safe but wouldn't pass the compiler's static borrow checker. The main example for this is getting a mutable reference to the interior of an object that was received immutably (which, under the statically enforced rules of Rust, would make the entire interior immutable).

The types `Ref<T>` and `RefMut<T>` are the runtime-checked equivalents of `&T` and `&mut T` respectively.

(EDIT: This whole thing is somewhat of a lie. `&mut` really means "unique borrow" and `&` means "non-unique borrow". Certain types, like mutexes, can be non-uniquely but still mutably borrowed, because otherwise they would be useless.)

------

Rust's ownership model tries to push you to write programs in which objects' lifetimes are known at compile time. This works well in certain scenarios, but makes other scenarios difficult or impossible to express.

`Rc<T>` and its atomic sibling `Arc<T>` are reference-counting wrappers of `T`. They offer you an alternative to the ownership model.

They are useful when you want to use and properly dispose an object, but it is not easy (or possible) to determine, at the moment you're writing the code, which specific variable should be the owner of that object (and therefore should take care of disposing it). Much like in C++, this means that there is no single owner of the object and that the object will be disposed by the last reference-counting wrapper that points to it.


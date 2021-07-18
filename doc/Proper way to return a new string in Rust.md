Proper way to return a new string in Rust

I just spent a week reading the Rust Book, and now I'm working on my first program, which returns the filepath to the system wallpaper:

```rust
pub fn get_wallpaper() -> &str {
    let output = Command::new("gsettings");
    // irrelevant code
    if let Ok(message) = String::from_utf8(output.stdout) {
        return message;
    } else {
        return "";
    }
}
```

I'm getting the error `expected lifetime parameter on &str` and I know Rust wants an input `&str` which will be returned as the output because any `&str` I create inside the function will be cleaned up immediately after the function ends.

I know I can sidestep the issue by returning a `String` instead of a `&str`, and many answers to similar questions have said as much. But I can also seemingly do this:

```rust
fn main() {
    println!("message: {}", hello_string(""));
}

fn hello_string(x: &str) -> &str {
    return "hello world";
}
```

to get a `&str` out of my function. Can someone explain to me why this is bad and why I should never do it? Or maybe it's not bad and okay in certain situations?

answer1

You [cannot return a `&str` if you've allocated the `String` in the function](https://stackoverflow.com/q/29428227/155423). There's further discussion about [why](https://stackoverflow.com/q/29781331/155423), as well as the fact that it's [not limited to strings](https://stackoverflow.com/q/32682876/155423). That makes your choice *much* easier: return the `String`.

> `String`s are heap-allocated and built to be mutable.

`String`s are heap-allocated because they have an unknown length. Since that allocation is solely owned by the `String`, that's what grants the ability to mutate the string.

> My function just returns a filepath for reference purposes, and I'd rather leave it up to the caller to decide if they need a heap-stored mutable string.

This isn't possible. Your function has performed an allocation. If you don't return the allocation to the caller, then the value must be deallocated to prevent memory leaks. If it was returned after deallocation, that would be an invalid reference, leading to memory safety violations.

> But I can also seemingly do this:
>
> ```rust
> fn hello_string(x: &str) -> &str {
>     return "hello world";
> }
> ```
>
> to get a `&str` out of my function. Can someone explain to me why this is bad and why I should never do it? Or maybe it's not bad and okay in certain situations?

It's not *bad*, it just doesn't really allow you to do what you want in your original case. That `"hello world"` is a `&'static str`, a string slice that's been stored inside the code of the program itself. It has a fixed length and is known to live longer than `main`.

The signature `fn hello_string(x: &str) -> &str` can be expanded to `fn hello_string<'a>(x: &'a str) -> &'a str`. This indicates that the resulting string slice must have the *same* lifetime as the input string. A static string will outlive *any* lifetime, so that's valid to substitute.

This would be useful for a function where the result is based on the input string only:

```rust
fn long_string(x: &str) -> &str {
    if x.len() > 10 {
        "too long"
    } else {
        x
    }
}
```

However, in your case, the function owns the `String`. If you attempted to return a reference to a `String`, completely unrelated to the input string:

```rust
fn hello_string(x: &str) -> &str {
    &String::from("hello world")
}
```

You'll run into the common error message "borrowed value does not live long enough". That's because the borrowed value only lives until the end of method, not as long as the input string slice. You can't "trick" the compiler (or if you can, that's a major bug).


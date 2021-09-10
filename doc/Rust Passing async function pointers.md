Rust: Passing async function pointers

Let's say we have this code:

```rust
fn inc(src: u32) -> u32 {
    src + 1
}

fn inc2(src: u32) -> u32 {
    src + 2
}

type Incrementer = fn(u32) -> u32;
fn increment_printer(inc: Incrementer) {
    println!("{}", inc(1));
}

fn main() {
    increment_printer(inc);
    increment_printer(inc2);
}
```

Two functions with the same signature and a 3rd one that accepts pointers for them. Running this code results in 2\n3 being printed.

But something similar won't compile:

```rust
use core::future::Future;

async fn inc(src: u32) -> u32 {
    src + 1
}

async fn inc2(src: u32) -> u32 {
    src + 2
}

type Incrementer = fn(u32) -> dyn Future<Output = u32>;
async fn increment_printer(inc: Incrementer) {
    println!("{}", inc(1).await);
}

fn main() {
    async {
        increment_printer(inc).await;
        increment_printer(inc2).await;
    }
}
18 |         increment_printer(inc).await;
   |                           ^^^ expected trait object `dyn Future`, found opaque type
   |
   = note: expected fn pointer `fn(_) -> (dyn Future<Output = u32> + 'static)`
                 found fn item `fn(_) -> impl Future {inc}`
```

I know that each async function has its own type, as mentioned in the error. Is it possible to force the compiler to forget about the concrete type and treat them as similar types?

Maybe it's possible to force an async function to return a `Box<Future<Output=u32>>`?

I don't want to give up async functions for convenience. Also I don't want to require calling Box::pin() before passing the async function pointer to the function.

It would be interesting to know if there are more options.

answer

> Maybe it's possible to force an async function to return a Box<Future<Output=u32>>?

Not really, but it's possible to wrap the function into a closure that calls the function and returns a boxed future. Of course, the closure itself needs to be boxed so functions like `increment_printer` can receive it, but both boxings can be encapsulated in a utility function. For example ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=d2156fdac1265b0c2e62bd31f9c356e6)):

```rust
use core::future::Future;
use std::pin::Pin;

async fn inc(src: u32) -> u32 {
    src + 1
}

async fn inc2(src: u32) -> u32 {
    src + 2
}

type Incrementer = Box<dyn FnOnce(u32) -> Pin<Box<dyn Future<Output = u32>>>>;

fn force_boxed<T>(f: fn(u32) -> T) -> Incrementer
where
    T: Future<Output = u32> + 'static,
{
    Box::new(move |n| Box::pin(f(n)))
}

async fn increment_printer(inc: Incrementer) {
    println!("{}", inc(1).await);
}

fn main() {
    async {
        increment_printer(force_boxed(inc)).await;
        increment_printer(force_boxed(inc2)).await;
    };
}
```

If you can afford to make functions like `increment_printer` generic, then you can get rid of the outer box ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=8435f34f94963c32a9c990be950cd27d)):

```rust
// emulate trait alias
trait Incrementer: FnOnce(u32) -> Pin<Box<dyn Future<Output = u32>>> {}
impl<T> Incrementer for T
    where T: FnOnce(u32) -> Pin<Box<dyn Future<Output = u32>>>
{
}

fn force_boxed<T>(f: fn(u32) -> T) -> impl Incrementer
where
    T: Future<Output = u32> + 'static,
{
    move |n| Box::pin(f(n)) as _
}

async fn increment_printer(inc: impl Incrementer) {
    println!("{}", inc(1).await);
}
```

> I don't want to give up async functions for convenience. Also I don't want to require calling `Box::pin()` before passing the async function pointer to the function.

You need to call `Box::pin()` *somewhere* - with the above the call is at least confined to one place, and you get an `Incrementer` type, of sorts, as the receiver of async functions passed through `force_boxed()`.

answer

> Maybe it's possible to force an async function to return a `Box<Future<Output=u32>>`?

You can do so by wrapping it in an ordinary (non-`async`) function:

```rust
fn boxed_inc(src: u32) -> Pin<Box<dyn Future<Output = u32>>> {
    Box::pin(inc(src))
}
// Use like:
increment_printer(boxed_inc).await;
```

However, that would quickly get tedious if you had to write `boxed_inc`, `boxed_inc2`, etc. for all the `async` functions that you needed to wrap. Here's a macro that wraps it on demand:

```rust
macro_rules! force_boxed {
    ($inc:expr) => {{
        // I think the error message referred to here is spurious, but why take a chance?
        fn rustc_complains_if_this_name_conflicts_with_the_environment_even_though_its_probably_fine(src: u32) -> Pin<Box<dyn Future<Output = u32>>> {
            Box::pin($inc(src))
        }
        rustc_complains_if_this_name_conflicts_with_the_environment_even_though_its_probably_fine
    }}
}
// Use like:
increment_printer(force_boxed!(inc)).await;
```

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=8435f34f94963c32a9c990be950cd27d)

This is similar to and inspired by the `force_boxed` function in [user4815162342's answer](https://stackoverflow.com/a/66769658/3650362), but in this case it has to be a macro in order to coerce the result to a function pointer instead of merely a closure.


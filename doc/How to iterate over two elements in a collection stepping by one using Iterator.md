How to iterate over two elements in a collection stepping by one using Iterator?

Suppose we have a vector:

```rust
let a = vec![1, 2, 3];
```

What is the best and shortest way to iterate over the elements so, that in the first iteration I receive a tuple of `(1, 2)`, and in the next iteration - `(2, 3)`, until there are no elements, so without producing the `(3, None)` or anything like that? It seems that `a.chunks(2)` is a bit different, it steps by two, while I need to step by one over every two consecutive elements in a collection.

answer1

There are multiple ways to do so:

**Using the standard library:**

**[`.zip`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.zip)**

```rust
let result = a.iter()
    .zip(a.iter().skip(1))
    .inspect(|(a, b)| println!("a: {}, b: {}", a, b)
    .collect::<Vec<_>>();
```

**[`.windows`](https://doc.rust-lang.org/std/primitive.slice.html#method.windows)**

```rust
let result = a.windows(2)
    .inspect(|w| println!("a: {}, b: {}", w[0], w[1]))
    .collect::<Vec<_>>();
```

Worth noting, in case this matters, is that `windows` iterates over subslices, so unlike the above method, `collect()` in this case will not give you a `Vec` of tuples.

**Using the `Itertools` crate:**

**[`.tuple_windows`](https://docs.rs/itertools/0.10.0/itertools/trait.Itertools.html#method.tuple_windows)**

```rust
use itertools::Itertools;

let result = a.iter()
    .tuple_windows()
    .inspect(|(a, b)| println!("a: {}, b: {}", a, b))
    .collect::<Vec<_>>();
```

All of the above methods will print the following:

```rust
a: 1, b: 2
a: 2, b: 3
```

------

As a bonus, `Itertools` as of recent also has a [`.circular_tuple_windows()`](https://docs.rs/itertools/0.10.0/itertools/trait.Itertools.html#method.circular_tuple_windows), which performs an extra iteration by including the last (`3`) and first (`1`) element:

```rust
use itertools::Itertools;

let result = a.iter()
    .circular_tuple_windows()
    .inspect(|(a, b)| println!("a: {}, b: {}", a, b))
    .collect::<Vec<_>>();
a: 1, b: 2
a: 2, b: 3
a: 3, b: 1
```

answer2

With the nightly-only (as of early 2021) `array_windows` feature, you can iterate over length-2 array references:

```rust
for [a, b] in arg.array_windows() {
    println!("a: {}, b: {}", a, b);
}
```

[Playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=300884fa6045cdcac995f0853ed68bd0)

Note the use of a slice pattern `[a, b]`, not a tuple pattern `(a, b)`.

On stable you can achieve the same thing with `windows` and `try_into`, but it's not as clean:

```rust
use std::convert::TryInto;

for s in arg.windows(2) {
    let [a, b]: [i32; 2] = s.try_into().unwrap();
    println!("a: {}, b: {}", a, b);
}
```


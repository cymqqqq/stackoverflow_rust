How to call count on an iterator and still use the iterator's items?

`parts.count()` leads to ownership transfer, so `parts` can't be used any more.

```rust
fn split(slice: &[u8], splitter: &[u8]) -> Option<Vec<u8>> {
    let mut parts = slice.split(|b| splitter.contains(b));

    let len = parts.count(); //ownership transfer

    if len >= 2 {
        Some(parts.nth(1).unwrap().to_vec())
    } else if len >= 1 {
        Some(parts.nth(0).unwrap().to_vec())
    } else {
        None
    }
}

fn main() {
    split(&[1u8, 2u8, 3u8], &[2u8]);
}
```

answer1

It is also possible to avoid unnecessary allocations of `Vec` if you only need to use the first or the second part:

```rust
fn split<'a>(slice: &'a [u8], splitter: &[u8]) -> Option<&'a [u8]> {
    let mut parts = slice.split(|b| splitter.contains(b)).fuse();

    let first = parts.next();
    let second = parts.next();

    second.or(first)
}
```

Then if you actually need a `Vec` you can map on the result:

```rust
split(&[1u8, 2u8, 3u8], &[2u8]).map(|s| s.to_vec())
```

Of course, if you want, you can move `to_vec()` conversion to the function:

```rust
second.or(first).map(|s| s.to_vec())
```

I'm calling [`fuse()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fuse) on the iterator in order to guarantee that it will always return `None` after the first `None` is returned (which is not guaranteed by the general iterator protocol).

answer2

One thing you can do is `collect` the results of the split in a new owned `Vec`, like this:

```rust
fn split(slice: &[u8], splitter: &[u8]) -> Option<Vec<u8>> {
    let parts: Vec<&[u8]> = slice.split(|b| splitter.contains(b)).collect();

    let len = parts.len();

    if len >= 2 {
        Some(parts.iter().nth(1).unwrap().to_vec())
    } else if len >= 1 {
        Some(parts.iter().nth(0).unwrap().to_vec())
    } else {
        None
    }
}
```

answer3

The other answers are good suggestions to answer your problem, but I'd like to point out another general solution: create multiple iterators:

```rust
fn split(slice: &[u8], splitter: &[u8]) -> Option<Vec<u8>> {
    let mut parts = slice.split(|b| splitter.contains(b));
    let parts2 = slice.split(|b| splitter.contains(b));

    let len = parts2.count();

    if len >= 2 {
        Some(parts.nth(1).unwrap().to_vec())
    } else if len >= 1 {
        Some(parts.nth(0).unwrap().to_vec())
    } else {
        None
    }
}

fn main() {
    split(&[1u8, 2u8, 3u8], &[2u8]);
}
```

You can usually create multiple read-only iterators. Some iterators even implement [`Clone`](http://doc.rust-lang.org/std/clone/trait.Clone.html), so you could just say `iter.clone().count()`. Unfortunately, [`Split`](http://doc.rust-lang.org/std/slice/struct.Split.html) isn't one of them because it owns the passed-in closure.


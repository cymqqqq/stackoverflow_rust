How does the Iterator::collect function work?

I am trying to understand the full capabilities of `collect` function by going through [some documentation](https://doc.rust-lang.org/std/iter/trait.Iterator.html#provided-methods). I am running into some challenges, specifically in the last example quoted on the page (also listed below, with my comments inline)

```rust
let results = [Ok(1), Err("nope"), Ok(3), Err("bad")];

let result: Result<Vec<_>, &str> = results.iter().cloned().collect();

// gives us the first error <-- Point 1
assert_eq!(Err("nope"), result);

let results = [Ok(1), Ok(3)];

let result: Result<Vec<_>, &str> = results.iter().cloned().collect();

// gives us the list of answers
assert_eq!(Ok(vec![1, 3]), result);
```

I followed up this code with some of my own (shown below)

```rust
let results: [std::result::Result<i32, &str>; 2] = [Err("nope"), Err("bad")];

let result: Vec<Result<i32, &str>> = results.iter().cloned().collect();

// The following prints <-- Point 2
// "nope"
// "bad"
for x in result{
    println!("{:?}", x.unwrap_err());
}
```

Looking at the implementation of [the `FromIterator` trait on the `Result` struct](https://doc.rust-lang.org/src/core/result.rs.html#283-288), we see that it mentions that "Takes each element in the `Iterator`: if it is an `Err`, no further elements are taken, and the `Err` is returned. Should no `Err` occur, a container with the values of each `Result` is returned.

This explanation is in line with the result that is seen at Point 1 but doesn't seem to work with Point 2. In point 2 I was expecting only "nope" to be printed instead of both values.

Hence, I am trying to understand where this (selective) conversion is happening and running into a challenge.

If we look at the method definition itself we see the following.

```rust
#[inline]
fn from_iter<I: IntoIterator<Item=Result<A, E>>>(iter: I) -> Result<V, E> {
    // FIXME(#11084): This could be replaced with Iterator::scan when this
    // performance bug is closed.

    iter::process_results(iter.into_iter(), |i| i.collect())
}
```

It shows that the `into_iter()` method is being called on the iterator. Searching for `into_iter` gives two implementations

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T, E> IntoIterator for Result<T, E> {
    type Item = T;
    type IntoIter = IntoIter<T>;

    /// Returns a consuming iterator over the possibly contained value.
    ///
    /// The iterator yields one value if the result is [`Result::Ok`], otherwise none.
    ///
    /// # Examples
    ///
    /// Basic usage:
    ///
    /// ```
    /// let x: Result<u32, &str> = Ok(5);
    /// let v: Vec<u32> = x.into_iter().collect();
    /// assert_eq!(v, [5]);
    ///
    /// let x: Result<u32, &str> = Err("nothing!");
    /// let v: Vec<u32> = x.into_iter().collect();
    /// assert_eq!(v, []);
    /// ```
    #[inline]
    fn into_iter(self) -> IntoIter<T> {
        IntoIter { inner: self.ok() }
    }
}

#[stable(since = "1.4.0", feature = "result_iter")]
impl<'a, T, E> IntoIterator for &'a Result<T, E> {
    type Item = &'a T;
    type IntoIter = Iter<'a, T>;

    fn into_iter(self) -> Iter<'a, T> {
        self.iter()
    }
}
```

However, in my limited understanding of the language none seem to be able to explain what the documentation says as well as what is happening in Point 2.

Could someone please explain how this is working or point me to the right place in source where such selection logic is implemented?

What I want to understand is not why we get all values in a vector and only one in the result, but a. where is the code/logic to select the first `Err` from a list of values and b. why are multiple `Err` values selected when the result is collected in a list (when according to the documentation it should be only the first `Err` value)

answer1

In this example

```rust
let result: Vec<Result<i32, &str>> = results.iter().cloned().collect();
```

you do not collect into a `Result`, but into a `Vec`, hence all values are collected, untouched. This is expected from `Vec`.

This is fundamentally different from

```rust
let result: Result<Vec<_>, &str> = results.iter().cloned().collect();
```

where you collect into a `Result`, which does filter elements depending on whether an `Err` is found or not. This comes from [`impl FromIterator> for Result where V: FromIterator,`](https://doc.rust-lang.org/std/result/enum.Result.html#impl-FromIterator>).


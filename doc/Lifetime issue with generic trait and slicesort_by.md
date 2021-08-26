Lifetime issue with generic trait and slice::sort_by

As a learning exercise, I've been writing a sorting library and I'm running into a roadblock. I've defined a trait `ExtractFrom` to extract a sortable key from items in a slice (to do the equivalent of what `sort_by_key` would do). I would like to be able to extract a key that borrows data, but my attempts to implement that have failed.

Here is a reduced example that demonstrates what I've attempted. `LargeData` is what is contained within the slice, and I've defined `LargeDataKey` that contains references to the subset of the data I want to sort by. This is running into lifetime issues between the `extract_from` implementation and what `sort_by` expects, but I don't know how to fix it. Any explanation or suggestions on how to best accomplish this would be appreciated.

```rust
trait ExtractFrom<'a, T> {
    type Extracted;
    fn extract_from(&'a T) -> Self::Extracted;
}

fn sort_by_extractor<'a, T, E>(vec: Vec<T>)
where
    E: ExtractFrom<'a, T>,
    E::Extracted: Ord,
{
    vec.sort_by(|a, b| {
        let ak = &E::extract_from(a);
        let bk = &E::extract_from(b);
        ak.cmp(bk)
    })
}

#[derive(Debug, PartialOrd, Ord, PartialEq, Eq)]
struct LargeData(String, String, String);

#[derive(Debug, PartialOrd, Ord, PartialEq, Eq)]
struct LargeDataKey<'a>(&'a str, &'a str);

impl<'a> ExtractFrom<'a, LargeData> for LargeDataKey<'a> {
    type Extracted = LargeDataKey<'a>;
    fn extract_from(input: &'a LargeData) -> LargeDataKey<'a> {
        LargeDataKey(&input.2, &input.0)
    }
}

fn main() {
    let v = vec![
        LargeData("foo".to_string(), "bar".to_string(), "baz".to_string()),
        LargeData("one".to_string(), "two".to_string(), "three".to_string()),
        LargeData("four".to_string(), "five".to_string(), "six".to_string()),
    ];
    sort_by_extractor::<LargeData, LargeDataKey>(v);
    println!("hello");
}
```

This code is also available on the [Rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=ad9c7f49d6a0a36b555cf00697c0de78).

This fails with:

```none
error[E0495]: cannot infer an appropriate lifetime for lifetime parameter `'a` due to conflicting requirements
  --> src/main.rs:12:19
   |
12 |         let ak = &E::extract_from(a);
   |                   ^^^^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #2 defined on the body at 11:17...
  --> src/main.rs:11:17
   |
11 |       vec.sort_by(|a, b| {
   |  _________________^
12 | |         let ak = &E::extract_from(a);
13 | |         let bk = &E::extract_from(b);
14 | |         ak.cmp(bk)
15 | |     })
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/main.rs:12:35
   |
12 |         let ak = &E::extract_from(a);
   |                                   ^
note: but, the lifetime must be valid for the lifetime 'a as defined on the function body at 6:22...
  --> src/main.rs:6:22
   |
6  | fn sort_by_extractor<'a, T, E>(vec: Vec<T>)
   |                      ^^
   = note: ...so that the types are compatible:
           expected ExtractFrom<'_, T>
              found ExtractFrom<'a, T>
```

answer1

Your code would more likely be written as

```rust
#[derive(Debug)]
struct LargeData(String, String, String);

#[derive(Debug, PartialOrd, Ord, PartialEq, Eq)]
struct LargeDataKey<'a>(&'a str, &'a str);

impl<'a> From<&'a LargeData> for LargeDataKey<'a> {
    fn from(input: &'a LargeData) -> LargeDataKey<'a> {
        LargeDataKey(&input.2, &input.0)
    }
}

fn main() {
    let mut v = vec![
        LargeData("foo".to_string(), "bar".to_string(), "baz".to_string()),
        LargeData("one".to_string(), "two".to_string(), "three".to_string()),
        LargeData("four".to_string(), "five".to_string(), "six".to_string()),
    ];
    v.sort_by_key(|x| LargeDataKey::from(x));
    println!("hello");
}
```

As [rodrigo suggests](https://stackoverflow.com/questions/53072850/lifetime-issue-with-generic-trait-and-slicesort-by/53084439#comment93066120_53072850), your code cannot be implemented in stable Rust 1.30. This is *why* `sort_by_key` has the limitation that it does: it's currently impossible to design a trait that covers your use case.

The problem is that Rust does not currently have the concept of [*generic associated types*](https://github.com/rust-lang/rfcs/blob/master/text/1598-generic_associated_types.md). This is needed to be able to define an associated type that has a constructor that can take in a late-bound lifetime.

You can instead use `sort_by` directly, so long as the returned types don't escape the closure:

```rust
v.sort_by(|a, b| {
    let a = LargeDataKey::from(a);
    let b = LargeDataKey::from(b);
    a.cmp(&b)
});
```

answer2

The compiler error clearly states that there are two lifetimes at play here:

```rust
vec.sort_by(|a: &T, b: &T| {
    let ak = &E::extract_from(a);
    let bk = &E::extract_from(b);
    ak.cmp(bk)
})
```

- The anonymous lifetime associated with `a: &T` and `b: &T` closure args
- The lifetime associated with the `'a` lifetime parameter (`fn extract_from(&'a T)`)

I did not find a way to get rid of this lifetime mismatch while maintaining your design.

If your goal is it to extract a sortable key from items in a slice, here's an approach that works based on implementing `Ord` for `LargeData`:

```rust
use std::cmp::Ordering;

#[derive(Debug, PartialOrd, PartialEq, Eq)]
struct LargeData(String, String, String);

// really needed?
// see impl in LargeData::cmp() below
#[derive(Debug, PartialOrd, Ord, PartialEq, Eq)]
struct LargeDataKey<'a>(&'a str, &'a str);

impl Ord for LargeData {
    fn cmp(&self, other: &LargeData) -> Ordering {
        //let op1 = LargeDataKey(&self.2, &self.0);
        //let op2 = LargeDataKey(&other.2, &other.0);
        //op1.cmp(&op2)
        (&self.2, &self.0).cmp(&(&other.2, &other.0))
    }
}

fn sort_by_extractor<E, T>(vec: &mut Vec<T>, extractor: E)
where
    E: FnMut(&T, &T) -> Ordering,
{
    vec.sort_by(extractor);
}

fn main() {
    let mut v = vec![
        LargeData("foo".to_string(), "bar".to_string(), "baz".to_string()),
        LargeData("one".to_string(), "two".to_string(), "three".to_string()),
        LargeData("four".to_string(), "five".to_string(), "six".to_string()),
    ];

    sort_by_extractor(&mut v, |a, b| a.cmp(b));
    println!("{:?}", v);
}
```


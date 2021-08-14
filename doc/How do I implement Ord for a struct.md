How do I implement Ord for a struct?

I've seen a question similar to this one, but no one that tells me exactly how to implement `Ord` for a struct. For example, the following:

```rust
struct SomeNum {
    name: String,
    value: u32,
}

impl Ord for SomeNum {
    fn cmp(&self, other:&Self) -> Ordering {
        let size1 = self.value;
        let size2 = other.value;
        if size1 > size2 {
            Ordering::Less
        }
        if size1 < size2 {
            Ordering::Greater
        }
        Ordering::Equal
    }
}
```

This gives me the error:

```rust
error: the trait `core::cmp::Eq` is not implemented for the type `SomeNum` [E0277]
```

How would I fix this? I have tried changing the implementation to:

```rust
impl Ord for SomeNum where SomeNum: PartialOrd + PartialEq + Eq {...}
```

and adding the appropriate `partial_cmp` and `eq` functions but it gives me the error that both those methods are not a member of `Ord`.

answer1

The definition of [`Ord`](http://doc.rust-lang.org/std/cmp/trait.Ord.html) is this:

```rust
pub trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;
}
```

Any type that implements `Ord` must also implement `Eq` and `PartialOrd<Self>`. You must implement these traits for `SomeNum`.

Incidentally, your implementation looks like being the wrong way round; if `self.value` is all you are comparing, `self.value > other.value` should be `Greater`, not `Less`.

You can use the `Ord` implementation on `u32` to assist, should you desire it: `self.value.cmp(other.value)`.

You should also take into account that `Ord` is a *total* ordering. If your `PartialEq` implementation, for example, takes `name` into consideration, your `Ord` implementation must also. It might be well to use a tuple for convenience (indicating that the most important field in the comparison is `value`, but that if they are the same, `name` should be taken into account), something like this:

```rust
struct SomeNum {
    name: String,
    value: u32,
}

impl Ord for SomeNum {
    fn cmp(&self, other: &Self) -> Ordering {
        (self.value, &self.name).cmp(&(other.value, &other.name))
    }
}

impl PartialOrd for SomeNum {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for SomeNum {
    fn eq(&self, other: &Self) -> bool {
        (self.value, &self.name) == (other.value, &other.name)
    }
}

impl Eq for SomeNum { }
```

If you’re doing it like this, you might as well reorder the fields and use `#[derive]`:

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct SomeNum {
    value: u32,
    name: String,
}
```

This will expand to basically the same thing.
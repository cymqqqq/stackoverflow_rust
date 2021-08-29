Extended Traits in Collections

I have a plain trait `Fruit` and an extended trait `WeightedFruit`. The Rust compiler accepts the `Fruit` trait in a `LinkedList` but not `WeightedFruit` in `BTreeSet`. What should be changed to make the sorted set work?

```rust
pub trait Fruit { }

pub trait WeightedFruit: Fruit + Ord { }

pub fn main() {
    let unsorted: LinkedList<Box<Fruit>> = LinkedList::new();
    let sorted: BTreeSet<Box<WeightedFruit>> = BTreeSet::new();
}
```

The error messages are:

```none
the trait `WeightedFruit` cannot be made into an object
trait `WeightedFruit: std::cmp::Ord` not satisfied
...
```

answer1

```rust
pub trait WeightedFruit: Fruit + Ord { }
```

This says the every struct that implements `WeightedFruit` must be comparable to itself. But not to other structs that implement that trait. So if an `Apple` implements `WeightedFruit`, it will be comparable to `Apple`, if an `Orange` implements `WeightedFruit`, it will be comparable to `Orange`, but not to each other.

You can not build a collection of "anything that is WeightedFruit", because they are not interchangeable - Apples and Oranges are different, because each is comparable to different kind.

Instead you want to do something like this:

```rust
use std::cmp::*;
use std::collections::*;

pub trait Fruit { }

pub trait WeightedFruit: Fruit {
    fn weight(&self) -> u32;
}

impl Ord for WeightedFruit {
    fn cmp(&self, other: &Self) -> Ordering {
        self.weight().cmp(&other.weight())
    }
}

impl PartialOrd for WeightedFruit {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for WeightedFruit {
    fn eq(&self, other: &Self) -> bool {
        self.weight() == other.weight()
    }
}

impl Eq for WeightedFruit {}


struct Apple {
    weight: u32
}

impl Fruit for Apple {}

impl WeightedFruit for Apple {
    fn weight(&self) -> u32 {
        self.weight
    }
}

struct Orange {
    weight: u32
}

impl Fruit for Orange {}

impl WeightedFruit for Orange {
    fn weight(&self) -> u32 {
        self.weight
    }
}

pub fn main() {
    let unsorted: LinkedList<Box<Fruit>> = LinkedList::new();
    let sorted: BTreeSet<Box<WeightedFruit>> = BTreeSet::new();
}
```

This says that every `WeightedFruit` fruit must be able to provide its `weight` and every `WeightedFruit` can be compared to any other `WeightedFruit` using this `weight`. Now you can make trait objects of `WeightedFruit` and mix them together in collections because they are interchangeable.

------

Additional explanation about the `Ord` and the `the trait ... cannot be made into an object` error:

When you work with trait objects, traits look kinda like interfaces in OO languages. You can have s trait and multiple structs that implement it and a function that takes a trait object. It can then call the functions of the trait on the object because it knows that it will have them and that they are exactly the same for every object. Just like in OO languages.

However traits have one extra feature: They can use the `Self` type in the function declarations. `Self` is always the type that implements the trait. If a trait uses the `Self` in any of its functions, it becomes special and can no longer be used as a trait object. Every time a struct implements such trait, it is implementing different version of it (a version where `Self` is different). You can not make a trait object because every struct that implements it is implementing different version of it.

The `Ord` in rust is like `Comparable<T>` in java, where `T` is selected for you by compiler. And just like you can not have a method that accepts anything `Comparable` in java (I hope you can not?), you can not have a method that accepts any `Ord` trait object.


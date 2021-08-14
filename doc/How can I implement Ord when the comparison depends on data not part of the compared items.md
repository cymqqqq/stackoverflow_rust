How can I implement Ord when the comparison depends on data not part of the compared items?

I have a small struct containing only an `i32`:

```rust
struct MyStruct {
   value: i32,
}
```

I want to implement `Ord` in order to store `MyStruct` in a `BTreeMap` or any other data structure that requires you to have `Ord` on its elements.

In my case, comparing two instances of `MyStruct` does not depend on the `value`s in them, but asking another data structure (a dictionary), and that data structure is unique for each instance of the `BTreeMap` I will create. So ideally it would look like this:

```rust
impl Ord for MyStruct {
    fn cmp(&self, other: &Self, dict: &Dictionary) -> Ordering {
        dict.lookup(self.value).cmp(dict.lookup(other.value))
    }
}
```

However this won't be possible, since an `Ord` implementation only can access two instances of `MyStruct`, nothing more.

One solution would be storing a pointer to the dictionary in `MyStruct` but that's overkill. `MyStruct` is supposed to be a simple wrapper and the pointer would double its size. Another solution is to use a static global, but that's not a good solution either.

In C++ the solution would be easy: Most STL algorithms/data structures let you pass a comparator, where it can be a function object with some state. So I believe Rust would have an idiom to match this somehow, is there any way to accomplish this?

answer1

I remember the debate over whether allowing a custom comparator was worth it or not, and it was decided that this complicated the API a lot when most of the times one could achieve the same effect by using a new (wrapping) type and redefine `PartialOrd` for it.

It was, ultimately, a trade-off: weighing API simplicity versus unusual needs (which are probably summed up as access to external resources).

------

In your specific case, there are two solutions:

- use the API the way it was intended: create a wrapper structure containing both an instance of `MyStruct` and a reference to the dictionary, then define `Ord` on that wrapper and use this as key in the `BTreeMap`
- circumvent the API... somehow

I would personally advise starting with using the API as intended, and measure, before going down the road of trying to circumvent it.

```rust
#[derive(Eq, PartialEq, Debug)]
struct MyStruct {
   value: i32,
}

#[derive(Debug)]
struct MyStructAsKey<'a> {
    inner: MyStruct,
    dict: &'a Dictionary,
}

impl<'a> Eq for MyStructAsKey<'a> {}

impl<'a> PartialEq for MyStructAsKey<'a> {
    fn eq(&self, other: &Self) -> bool {
        self.inner == other.inner && self.dict as *const _ as usize == other.dict as *const _ as usize
    }
}

impl<'a> Ord for MyStructAsKey<'a> {
    fn cmp(&self, other: &Self) -> ::std::cmp::Ordering {
        self.dict.lookup(&self.inner).cmp(&other.dict.lookup(&other.inner))
    }
}

impl<'a> PartialOrd for MyStructAsKey<'a> {
    fn partial_cmp(&self, other: &Self) -> Option<::std::cmp::Ordering> {
        Some(self.dict.lookup(&self.inner).cmp(&other.dict.lookup(&other.inner)))
    }
}

#[derive(Default, Debug)]
struct Dictionary(::std::cell::RefCell<::std::collections::HashMap<i32, u64>>);

impl Dictionary {
    fn ord_key<'a>(&'a self, ms: MyStruct) -> MyStructAsKey<'a> {
        MyStructAsKey {
            inner: ms,
            dict: self,
        }
    }
    fn lookup(&self, key: &MyStruct) -> u64 {
        self.0.borrow()[&key.value]
    }
    fn create(&self, value: u64) -> MyStruct {
        let mut map = self.0.borrow_mut();
        let n = map.len();
        assert!(n as i32 as usize == n);
        let n = n as i32;
        map.insert(n, value);
        MyStruct {
            value: n,
        }
    }
}

fn main() {
    let dict = Dictionary::default();
    let a = dict.create(99);
    let b = dict.create(42);
    let mut set = ::std::collections::BTreeSet::new();
    set.insert(dict.ord_key(a));
    set.insert(dict.ord_key(b));
    println!("{:#?}", set);
    let c = dict.create(1000);
    let d = dict.create(0);
    set.insert(dict.ord_key(c));
    set.insert(dict.ord_key(d));
    println!("{:#?}", set);
}
```


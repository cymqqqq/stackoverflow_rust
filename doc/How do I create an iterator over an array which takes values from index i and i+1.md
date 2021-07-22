How do I create an iterator over an array which takes values from index i and i+1?

I created a small example that runs in the Rust Playground:

```rust
#[derive(Clone)]
pub struct Person {
    pub firstname: Vec<u8>,
    pub surname: Vec<u8>,
}

fn main() {
    let p0 = Person {
        firstname: vec![0u8; 5],
        surname: vec![1u8; 5],
    };
    let p1 = Person {
        firstname: vec![2u8; 7],
        surname: vec![3u8; 2],
    };
    let p2 = Person {
        firstname: vec![4u8; 8],
        surname: vec![5u8; 8],
    };
    let p3 = Person {
        firstname: vec![6u8; 3],
        surname: vec![7u8; 1],
    };

    let people = [p0, p1, p2, p3];

    for i in 0..people.len() {
        if i + 1 < people.len() {
            println!(
                "{:?}",
                (people[i].firstname.clone(), people[i + 1].surname.clone())
            )
        }
    }
}
```

Given the array `people`, I want to iterate its elements and collect tuples of the first name of the `person` at index `i` and the surname of the person at index `i+1`. This simple for loop does the job, but let's assume that instead of `println` I would like to pass such tuple into some function `f`. I can easily do this in this `for` loop, but I would like to learn whether I can implement that using an iterator `iter()` (and later apply `collect()` or `fold` functions if needed) instead of using the `for` loop?

answer1

You can combine two features:

1. Create an array iterator that skips the first elements (`iter().skip(1)`).
2. [`Iterator.zip`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.zip)

```rust
#[derive(Clone)]
pub struct Person {
    pub firstname: Vec<u8>,
    pub surname: Vec<u8>,
}

fn main() {
    let p0 = Person {
        firstname: vec![0u8; 5],
        surname: vec![1u8; 5],
    };
    let p1 = Person {
        firstname: vec![2u8; 7],
        surname: vec![3u8; 2],
    };
    let p2 = Person {
        firstname: vec![4u8; 8],
        surname: vec![5u8; 8],
    };
    let p3 = Person {
        firstname: vec![6u8; 3],
        surname: vec![7u8; 1],
    };

    let people = [p0, p1, p2, p3];

    for (i, j) in people.iter().zip(people.iter().skip(1)) {
        println!("{:?}", (&i.firstname, &j.surname));
    }
}
```

[Permalink to the playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=1dc5806cb4b0862d4789cc6d76c0ef87)

answer2

I would use [`slice::windows`](https://doc.rust-lang.org/std/primitive.slice.html#method.windows):

```rust
for window in people.windows(2) {
    println!("{:?}", (&window[0].firstname, &window[1].surname))
}
```


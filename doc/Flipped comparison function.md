How to sort a Vector in descending order in Rust?

In Rust, the sorting methods of a `Vec` always arrange the elements from smallest to largest. What is a general-purpose way of sorting from largest to smallest instead?

If you have a vector of numbers, you can provide a key extraction function that "inverts" the number like this:

```rust
let mut numbers: Vec<u32> = vec![100_000, 3, 6, 2];
numbers.sort_by_key(|n| std::u32::MAX - n);
```

But that's not very clear, and it's not straightforward to extend that method to other types like strings.

answer1

There are at least three ways to do it.

# Flipped comparison function

```rust
vec.sort_by(|a, b| b.cmp(a))
```

This switches around the order in which elements are compared, so that smaller elements appear larger to the sorting function and vice versa.

# Wrapper with reverse Ord instance

```rust
use std::cmp::Reverse;
vec.sort_by_key(|w| Reverse(*w));
```

[`Reverse`](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) is a generic wrapper which has an `Ord` instance that is the opposite of the wrapped type's ordering.

If you try to return a `Reverse` containing a reference by removing the `*`, that results in a lifetime problem, same as when you return a reference directly inside `sort_by_key` (see also [this question](https://stackoverflow.com/questions/47121985/why-cant-i-use-a-key-function-that-returns-a-reference-when-sorting-a-vector-wi)). Hence, this code snippet can only be used with vectors where the keys are `Copy` types.

# Sorting then reversing

```rust
vec.sort();
vec.reverse();
```

It initially sorts in the wrong order and then reverses all elements.

# Performance

I benchmarked the three methods with `criterion` for a length 100_000 `Vec<u64>`. The timing results are listed in the order above. The left and right values show the lower and upper bounds of the confidence interval respectively, and the middle value is `criterion`'s best estimate.

Performance is comparable, although the flipped comparison function seems to be a tiny bit slower:

```rust
Sorting/sort_1          time:   [6.2189 ms 6.2539 ms 6.2936 ms]
Sorting/sort_2          time:   [6.1828 ms 6.1848 ms 6.1870 ms]
Sorting/sort_3          time:   [6.2090 ms 6.2112 ms 6.2138 ms]
```

To reproduce, save the following files as `benches/sort.rs` and `Cargo.toml`, then run `cargo bench`. There is an additional benchmark in there which checks that the cost of cloning the vector is irrelevant compared to the sorting, it only takes a few microseconds.

```rust
fn generate_shuffled_data() -> Vec<u64> {
    use rand::Rng;
    let mut rng = rand::thread_rng();
    (0..100000).map(|_| rng.gen::<u64>()).collect()
}

pub fn no_sort<T: Ord>(vec: Vec<T>) -> Vec<T> {
    vec
}

pub fn sort_1<T: Ord>(mut vec: Vec<T>) -> Vec<T> {
    vec.sort_by(|a, b| b.cmp(a));
    vec
}

pub fn sort_2<T: Ord + Copy>(mut vec: Vec<T>) -> Vec<T> {
    vec.sort_by_key(|&w| std::cmp::Reverse(w));
    vec
}

pub fn sort_3<T: Ord>(mut vec: Vec<T>) -> Vec<T> {
    vec.sort();
    vec.reverse();
    vec
}

use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn comparison_benchmark(c: &mut Criterion) {
    let mut group = c.benchmark_group("Sorting");
    let data = generate_shuffled_data();

    group.bench_function("no_sort", |b| {
        b.iter(|| black_box(no_sort(data.clone())))
    });

    group.bench_function("sort_1", |b| {
        b.iter(|| black_box(sort_1(data.clone())))
    });

    group.bench_function("sort_2", |b| {
        b.iter(|| black_box(sort_2(data.clone())))
    });

    group.bench_function("sort_3", |b| {
        b.iter(|| black_box(sort_3(data.clone())))
    });

    group.finish()
}

criterion_group!(benches, comparison_benchmark);
criterion_main!(benches);
[package]
name = "sorting_bench"
version = "0.1.0"
authors = ["nnnmmm"]
edition = "2018"

[[bench]]
name = "sort"
harness = false

[dev-dependencies]
criterion = "0.3"
rand = "0.7.3"
```


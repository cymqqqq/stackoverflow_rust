How do I add two Rust arrays element-wise?

One simple approach is to generate indices using the `enumerate` iterator method and fill `z` the "obvious" way, by assigning into the obtained indices:

```rust
type Coord = [i64; 3];
fn add(a: Coord, b: Coord) -> Coord {
    let mut z: Coord = [0, 0, 0];
    for (i, (aval, bval)) in a.iter().zip(&b).enumerate() {
        z[i] = aval + bval;
    }
    z
}
fn main() {
    let x: Coord = [1, 2, 3];
    let y: Coord = [1, 1, 1];
    assert!(add(x, y) == [2, 3, 4]);
}
```

In Rust, we can do better than that by noticing that `iter()` produces *references* into the array, and `iter_mut()` is available to produce mutable references. This results in code very similar to what you attempted to write:

```rust
fn add(a: Coord, b: Coord) -> Coord {
    let mut z: Coord = [0, 0, 0];
    for ((zref, aval), bval) in z.iter_mut().zip(&a).zip(&b) {
        *zval = aval + bval;
    }
    z
}
```

If this pattern of writing into `z` recurs with different operations, you can abstract the creation of new `Coord` and filling it with data into a generic function:

```rust
fn new_coord_from<F: Iterator<Item = i64>>(src: F) -> Coord {
    let mut result = [0; 3];
    for (rref, val) in result.iter_mut().zip(src) {
        *rref = val;
    }
    result
}
```

`add` then looks just like we'd like it to:

```rust
fn add(a: Coord, b: Coord) -> Coord {
    new_coord_from(a.iter().zip(&b).map(|(a, b)| a + b))
}
```

If I want to declare a function `fn cmap(a: Coord, b: Coord, f: F) -> Coord { new_coord_from(a.iter().zip(&b).map(f)); }` to abstract out the iter/zip part too so I can declare `fn add(a: Coord, b: Coord) -> Coord { cmap(a,b, |(a,b)| a + b) }` - what type do I need to put for F? 

It would be `cmap<F: Fn((&i64, &i64)) -> i64>(a: Coord, b: Coord, f: F) -> Coord`. I would prefer, however, to get rid of the references and have `F` receive two numbers instead of a tuple: `fn cmap<F: Fn(i64, i64) -> i64>(a: Coord, b: Coord, f: F) -> Coord { new_coord_from(a.iter().zip(&b).map(|(x, y)| f(*x, *y))) }`. The definition of `add` then boils down to the simplest form, `cmap(a, b, |x, y| x + y)` or even `cmap(a, b, i64::add)` (after bringing the `Add` trait into the scope with `use std::ops::Add`)
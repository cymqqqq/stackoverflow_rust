How do I create an empty mutable two dimensional array in Rust?

create a dynamically-sized 2D vector like this:

```rust
fn example(width: usize, height: usize) {
    // Base 1d array
    let mut grid_raw = vec![0; width * height];

    // Vector of 'width' elements slices
    let mut grid_base: Vec<_> = grid_raw.as_mut_slice().chunks_mut(width).collect();

    // Final 2d array `&mut [&mut [_]]`
    let grid = grid_base.as_mut_slice();

    // Accessing data
    grid[0][0] = 4;
}
```

create a 2D array like this (using `Vec`) if you don't have a known size at compile time:

```rust
let width = 4;
let height = 4;

let mut array = vec![vec![0; width]; height];
```

Use it like this:

```rust
array[2][2] = 5;

println!("{:?}", array);
```

**Initialization:**
There are several approaches for 2D Array initialization:

1. Using constants for M (rows) & N (columns)

```rust
const M: usize = 2;
const N: usize = 4;

let mut grid = [[0 as u8; N] ; M];
```

2 Explicit declaration with type annotations

```rust
let mut grid: [[u8; 4]; 2] = [[0; 4]; 2];
```

**Traversing:**
The **read-only** traversing is as easy as:

```rust
for (i, row) in grid.iter().enumerate() {
    for (j, col) in row.iter().enumerate() {
        print!("{}", col);
    }
    println!()
}
```

or

```rust
for el in grid.iter().flat_map(|r| r.iter()) {
    println!("{}", el);
}
```

**Updating element(s):**

```rust
for (i, row) in grid.iter_mut().enumerate() {
    for (j, col) in row.iter_mut().enumerate() {
        col = 1;
    }
}
```

two-dimensional string array:

```rust
fn main() {
    let width = 2;
    let height = 3;

    let mut a: Vec<Vec<String>> = vec![vec![String::from(""); width]; height];

    for i in 0..height {
        for j in 0..width {
            let s = format!("{}:{}", i + 1, j + 1);
            a[i][j] = s;
        }
    }
    println!("{:?}", a);
}
```

Output:

```rust
[
  ["1:1", "1:2"],
  ["2:1", "2:2"],
  ["3:1", "3:2"]
]
```

Here is the code snippet to create a two-dimensional vector and then fill it with user input:

```rust
use std::io;

fn main(){
    let width = 4;
    let height = 4;

    let mut array = vec![vec![0; width]; height];

    for i in 0..4 {
        let mut xstr = String::from("");
        io::stdin().read_line(&mut xstr).ok().expect("read error");
        array[i] = xstr
            .split_whitespace()
            .map(|s| s.parse().expect("parse error"))
            .collect();
    }

    println!("{:?}", array)
}
```
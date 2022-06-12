Coupled arrays in Rust

I'm trying to couple 2 arrays to one "parent array". I'd like to have one main array like this for example:

```rust
let mut board: [[u8; 4]; 4] = [[1, 6, 5, 2],
                               [4, 8, 9, 3],
                               [9, 2, 2, 5], 
                               [3, 7, 6, 7]];
```

And 2 other arrays, one for the columns, and one for the 2*2 squares. The two other arrays should update when I change something in the main array.

The two other arrays would look like this in the example:

```rust
columns
[[1, 4, 9, 3],
 [6, 8, 2, 7],
 [5, 9, 2, 6],
 [2, 3, 5, 7]]

2*2 squares
[[1, 6, 4, 8],
 [5, 2, 9, 3],
 [9, 2, 3, 7],
 [2, 5, 6, 7]]
```

now if i say ` board[0][0] = 5;` the columns array should now look like this:

```rust
[[5, 4, 9, 3],
 [6, 8, 2, 7],
 [5, 9, 2, 6],
 [2, 3, 5, 7]]
```

Is there any way to do something like this in Rust?

Newtype pattern with a related trait implementations might be a solution. These should maps indicies to the base array, e.g. for your first case immutable access can be done via [`std::ops::Index`](https://doc.rust-lang.org/std/ops/trait.Index.html) ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=418af40adf874cd8701c048728026812)):

```rust
struct Board([u8; 4]; 4]);
impl Board {
    fn column(&self) -> ColumnBoard {
        ColumnBoard(self)
    }
}
struct ColumnBoard<'a>(&'a Board);
impl<'a> std::ops::Index<(usize, usize)> for ColumnBoard<'a> {
    type Output = u8;
    fn index(&self, (x, y): (usize, usize)) -> &Self::Output {
        &(self.0).0[y][x]
    }
}
fn main() {
    let board = Board([[1, 6, 5, 2],
                       [4, 8, 9, 3],
                       [9, 2, 2, 5], 
                       [3, 7, 6, 7]]);
                       
    let column = board.column();
    assert_eq!(column[(0, 2)], 9);
}
```

Similarly you can do for the second case. If you want to have an indexing operation twice, you might change `Output` to temporary structure which also implements `std::ops::Index`. For mutability, you need additionally implement [`std::ops::IndexMut`](https://doc.rust-lang.org/std/ops/trait.IndexMut.html) ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=87a25f7e5bc9ad60b088a47cf51595bb)):

```rust
impl<'a> std::ops::IndexMut<(usize, usize)> for ColumnBoard<'a> {
    fn index_mut(&mut self, (x, y): (usize, usize)) -> &mut Self::Output {
        &mut (self.0).0[y][x]
    }
}

// besides structures must explicitly declare mutability:

impl Board {
    fn column(&mut self) -> ColumnBoard {
        ColumnBoard(self)
    }
}

struct ColumnBoard<'a>(&'a mut Board);
```
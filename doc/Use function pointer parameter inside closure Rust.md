Use function pointer parameter inside closure Rust

I am looking to create a function, `agg`, which takes, as a parameter, another function, `get_id`, and returns an `FnMut` closure that uses the `get_id` function.

Concrete example:

```rust
struct CowRow {
    pub id : i32,
}
impl CowRow {
    fn get_id(&self) -> i32       { self.id }
}

pub fn agg<F>(col: F) -> Box<FnMut(&CowRow) -> i32>
    where F: Fn(&CowRow) -> i32 {
    let mut res = 0;
    Box::new(move |r| { res += col(&r); return res })
}

fn main() {
    let mut cow = CowRow { id: 0 };
    let a = agg(CowRow::get_id);
    a(&cow);
```

Which produces the error:

```rust
the parameter type `F` may not live long enough [E0310]

run `rustc --explain E0310` to see a detailed explanation

consider adding an explicit lifetime bound `F: 'static`...

...so that the type `[closure@main.rs:23:14: 23:53 col:F, res:i32]` will meet its required lifetime bounds
```

The idea here is that I want a generic function that allows for creating closures which operate on different fields in the struct. So, my thought was to pass a function in that is a getter for the struct and use this in the closure to extract the appropriate field.

I've tried various combinations of adding `'static` to the `agg` signature but I'm not sure what that actually means and where it would need to go syntactically. Additionally, I've tried a number of techniques from: https://github.com/nrc/r4cppp/blob/master/closures.md such as adding the `get_id` method as a trait but have been unable to get that working either.

answer

The type parameter `F` to your function has an associated lifetime (just like every other type). But implicitly, the return value of your function, `Box<FnMut(&CowRow) -> i32>`, is really `Box<FnMut(&CowRow) -> i32 + 'static>`. That is, unless you specify a lifetime for a box, it assumes its contents can live forever. Of course if `F` only lives for `'a`, then the borrow checker is going to complain. To fix this, either

- Force `F` to have a static lifetime so that it can live forever inside the box ([playpen](https://play.rust-lang.org/?gist=16dfb91e4ef1ea63e70866b1c0a07d34&version=stable&backtrace=1)):

  ```rust
  fn agg<F>(col: F) -> Box<FnMut(&CowRow) -> i32>
      where F: Fn(&CowRow) -> i32 + 'static
  {
      ...
  }
  ```

- Explicitly state that `F` has lifetime `'a` and so does the `Box` ([playpen](https://play.rust-lang.org/?gist=fc4abdc705185e852401cc63a81ea017&version=stable&backtrace=1)):

  ```rust
  fn agg<'a, F>(col: F) -> Box<FnMut(&CowRow) -> i32 + 'a>
      where F: Fn(&CowRow) -> i32 + 'a
  {
      ...
  }
  ```

The second version is more general than the first and will accept more closures as arguments.
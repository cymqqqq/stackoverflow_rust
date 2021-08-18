How to impl add for a generic in Rust

Why does the compiler not know how to add &Point ? Getting the following error: error[E0369]: cannot add `&Point<TextP>` to `&Point<TextP>` --> src/main.rs:72:16 | 72 | let g = &e + &f; | -- ^ -- &Point | | | &Point | = note: an implementation of `std::ops::Add` might be missing for `&Point<TextP>`

```rust
use std::ops::Add;
use std::cmp::PartialEq;

#[derive(Debug)]
pub struct Point<T> {
    x:Option<T>,
    y:T,
}

#[derive(Debug)]
pub struct TextP {
    p: i32
}

impl Add for TextP {
    type Output = TextP;
    fn add(self, other: TextP) -> TextP {
        TextP{p: self.p + other.p}
    }
}

impl <'a, 'b> Add<&'b TextP> for &'a TextP {
    type Output = TextP;
    fn add(self, other: &'b TextP) -> TextP {
        TextP {p: self.p + other.p}
    }
} 


impl<'a, 'b, T> Add<&'b Point<T>> for &'a Point<T> 
    where &'a T: Add<&'b T, Output=T>, 
      T: PartialEq + Add<Output = T>
{
    type Output = Point<T>;
    fn add(self, other: &'b Point<T>) -> Point<T> {
        Point {
            x: Some(self.x.as_ref().unwrap() + other.x.as_ref().unwrap()),
            y: self.y + other.y
        }
    }
}




impl<T: Add<Output = T>> Add for Point<T> {
    type Output = Point<T>;
    fn add(self, other: Point<T>) -> Point<T> {
        Point {
            x:Some(self.x.unwrap() + other.x.unwrap()),
            y:self.y + other.y,
        }
    }
}



fn main() {
    let a = Point{x:Some(5.5), y:5.5};
    let b = Point{x:Some(30.5), y:50.6};
    let c = &a + &b;
    println!("{:?}", a);
    let d = a + b;
    println!("c = {:?}",c);
    println!("d = {:?}",d);

    let e = Point{x:Some(TextP{p:1}), y:TextP{p:2}};
    let f = Point{x:Some(TextP{p:1}), y:TextP{p:2}};
    let g = &e + &f;
    println!("{:?}", g);

    let x = TextP {p: 1};
    let y = TextP {p: 1};
    let z = &x + &y;
}
```

answer

You're requiring `T` to be `PartialEq` in your `Add` instance, even though you don't supply `PartialEq` for `TextP`, nor do you need it. Just remove the extraneous constraint.

```rust
impl<'a, 'b, T> Add<&'b Point<T>> for &'a Point<T> 
where &'a T: Add<&'b T, Output=T> {
  ...
}
```

Next, you don't have ownership over the `T` values inside the `Point` value (since you only borrowed the `Point` itself), so all you can do is add the references, not the values.

```rust
impl<'a, 'b, T> Add<&'b Point<T>> for &'a Point<T> 
where &'a T: Add<&'b T, Output=T> {
    type Output = Point<T>;
    fn add(self, other: &'b Point<T>) -> Point<T> {
        Point {
            x: Some(self.x.as_ref().unwrap() + other.x.as_ref().unwrap()),
            y: &self.y + &other.y
        }
    }
}
```


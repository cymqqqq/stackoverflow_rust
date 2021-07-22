Writing an iterator

I'm writing a Vector struct in Rust.

```rust
pub struct Vector {
    pub x: f32,
    pub y: f32,
    pub z: f32,

    curr: uint
}
```

And I'd like to write a simple iterator for it, so that I can iterate over the elements of the vector. It's occasionally useful, plus I know next to nothing about iterators in Rust.

Here's what I've got at the moment.

```rust
impl Iterator<f32> for Vector {
    fn next(&mut self) -> Option<f32> {
        let new_next : Option<f32> = match self.curr {
            0 => Some(self.x),
            1 => Some(self.y),
            2 => Some(self.z), 
            _ => None
        };
        let new_curr = (self.curr + 1) % 4;
        mem::replace(&mut self.curr, new_curr);
        new_next
    }
}
```

Now ideally I'd like to be able to use this like:

```rust
let u = Vector::new(0.0f32, 0.0f32, 0.0f32);
for element in u {
    ///
}
```

However, I get the following compiler error:

```rust
 error: cannot borrow immutable local variable `u` as mutable
```

So I'm stumped. After a couple hours of Googling, I couldn't come up with anything. I feel like I'm missing something huge.

answer1

Are you sure you really want the `Vector` *itself* to be an iterator? Usually structures and iterators into them are separate. Consider something like this:

```rust
pub struct Vector {
    pub x: f32,
    pub y: f32,
    pub z: f32,
}

pub struct VectorIter<'a> {
    vector: &'a Vector,
    cur: usize,
}

impl<'a> Iterator for VectorIter<'a> {
    type Item = f32;

    fn next(&mut self) -> Option<f32> {
        let r = match self.cur {
            0 => self.vector.x,
            1 => self.vector.y,
            2 => self.vector.z,
            _ => return None,
        };
        self.cur += 1;
        Some(r)
    }
}

impl Vector {
    fn iter(&self) -> VectorIter {
        VectorIter {
            vector: self,
            cur: 0,
        }
    }
}

fn main() {
    let v = Vector { x: 1.0, y: 2.0, z: 3.0 };
    for c in v.iter() {
        println!("{}", c);
    }
}
```

Because `Vector` is very simple, it can derive `Copy`, and its iterator can take it by value:

```rust
#[derive(Copy, Clone)]
pub struct Vector {
    pub x: f32,
    pub y: f32,
    pub z: f32,
}

pub struct VectorIter {
    vector: Vector,
    cur: usize,
}

impl Iterator for VectorIter {
    type Item = f32;
    fn next(&mut self) -> Option<f32> {
        let r = match self.cur {
            0 => self.vector.x,
            1 => self.vector.y,
            2 => self.vector.z,
            _ => return None,
        };
        self.cur += 1;
        Some(r)
    }
}

impl Vector {
    fn iter(&self) -> VectorIter {
        VectorIter {
            vector: *self,
            cur: 0,
        }
    }
}

fn main() {
    let v = Vector { x: 1.0, y: 2.0, z: 3.0 };
    for c in v.iter() {
        println!("{}", c);
    }
}
```

This variant is probably better unless your `Vector` contains something other than coordinates. This variant is more flexible because it does not tie the iterator with the iterable, but on the other hand, exactly because of the same reason it may be undesirable (with `Copy` you can change the original value, and the iterator won't reflect it; without `Copy` and with references you won't be able to change the original value at all). The semantics you would want to use heavily depends on your use cases.

answer2

> Editor's note: This answer is no longer useful as of Rust 1.0 due to changes how the `for` loop works.

Vladimir Matveev's answer is correct and explains what you should do. I'd like to add on by explaining your error a bit more.

> error: cannot borrow immutable local variable `u` as mutable

When you are iterating over something, you need to mutate *something* that tracks where you are. In your code, that's done by:

```rust
mem::replace(&mut self.curr, new_curr);
```

And Rust knows which thing you want to mutate because the method signature for `next` indicates it:

```rust
fn next(&mut self) -> Option<f32>
```

The problem is that your object *is not mutable*:

```rust
let u = Vector::new(0.0f32, 0.0f32, 0.0f32);
```

If you change your code to

```rust
let mut u = Vector::new(0.0f32, 0.0f32, 0.0f32);
```

I'd expect your code to work as-is. All that being said, a separate iterator is probably the right way to go. However, you'll know more than we do about your code!


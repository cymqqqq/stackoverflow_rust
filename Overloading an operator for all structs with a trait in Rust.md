Overloading an operator for all structs with a trait in Rust

I'm trying to implement C++-style expression templates in Rust using traits and operator overloading. I'm getting stuck trying to overload '+' and '*' for every expression template struct. The compiler complains about the `Add` and `Mul` trait implementations:

```none
error[E0210]: type parameter `T` must be used as the type parameter for some local type (e.g., `MyStruct<T>`)
  --> src/main.rs:32:6
   |
32 | impl<T: HasValue + Copy, O: HasValue + Copy> Add<O> for T {
   |      ^ type parameter `T` must be used as the type parameter for some local type
   |
   = note: implementing a foreign trait is only possible if at least one of the types for which it is implemented is local
   = note: only traits defined in the current crate can be implemented for a type parameter
```

That error would make sense if the type I was trying to implement a trait for was constructible without my crate, but the type is a generic that must implement the `HasValue` trait I defined.

Here's the code:

```rust
use std::ops::{Add, Mul};

trait HasValue {
    fn get_value(&self) -> i32;
}

// Val

struct Val {
    value: i32,
}

impl HasValue for Val {
    fn get_value(&self) -> i32 {
        self.value
    }
}

// Add

struct AddOp<T1: HasValue + Copy, T2: HasValue + Copy> {
    lhs: T1,
    rhs: T2,
}

impl<T1: HasValue + Copy, T2: HasValue + Copy> HasValue for AddOp<T1, T2> {
    fn get_value(&self) -> i32 {
        self.lhs.get_value() + self.rhs.get_value()
    }
}

impl<T: HasValue + Copy, O: HasValue + Copy> Add<O> for T {
    type Output = AddOp<T, O>;
    fn add(&self, other: &O) -> AddOp<T, O> {
        AddOp {
            lhs: *self,
            rhs: *other,
        }
    }
}

// Mul

struct MulOp<T1: HasValue + Copy, T2: HasValue + Copy> {
    lhs: T1,
    rhs: T2,
}

impl<T1: HasValue + Copy, T2: HasValue + Copy> HasValue for MulOp<T1, T2> {
    fn get_value(&self) -> i32 {
        self.lhs.get_value() * self.rhs.get_value()
    }
}

impl<T: HasValue + Copy, O: HasValue + Copy> Mul<O> for T {
    type Output = MulOp<T, O>;
    fn mul(&self, other: &O) -> MulOp<T, O> {
        MulOp {
            lhs: *self,
            rhs: *other,
        }
    }
}

fn main() {
    let a = Val { value: 1 };
    let b = Val { value: 2 };
    let c = Val { value: 2 };

    let e = ((a + b) * c).get_value();

    print!("{}", e);
}
```

Thoughts?

answer1

Trying to define the trait `Add` for you custom types, you are doing this:

```rust
impl<T: HasValue + Copy, O: HasValue + Copy> Add<O> for T {
    type Output = AddOp<T, O>;
    fn add(&self, other: &O) -> AddOp<T, O> {
        AddOp {
            lhs: *self,
            rhs: *other,
        }
    }
}
```

But `T: HasValue + Copy` matches any type implementing trait `HasValue`, and this type may not be defined in your crate (for example if you implement `HasValue` for `i32`). As `Add` isn't defined in your crate either, Rust complains: for example by defining `HasValue` for `i32`, you would also re-define `Add<i32>` for `i32`!

My suggestion would be to wrap all your operation and values structs into a generic one, and implementing `Add` and `Mul` for it. This way, you implement `Add` and `Mul` only for a simple type defined in your crate, and the compiler is happy.

Something like that:

```rust
struct Calculus<T> {
    calc: T,
}

impl<T: HasValue + Copy> HasValue for Calculus<T> {
    fn get_value(&self) -> i32 {
        self.calc.get_value()
    }
}

impl<T, O> Add<Calculus<O>> for Calculus<T>
where
    T: HasValue + Copy,
    O: HasValue + Copy,
{
    type Output = Calculus<AddOp<T, O>>;
    fn add(self, other: Calculus<O>) -> Calculus<AddOp<T, O>> {
        Calculus {
            calc: AddOp {
                lhs: self.calc,
                rhs: other.calc,
            },
        }
    }
}

impl<T, O> Mul<Calculus<O>> for Calculus<T>
where
    T: HasValue + Copy,
    O: HasValue + Copy,
{
    type Output = Calculus<MulOp<T, O>>;
    fn mul(self, other: Calculus<O>) -> Calculus<MulOp<T, O>> {
        Calculus {
            calc: MulOp {
                lhs: self.calc,
                rhs: other.calc,
            },
        }
    }
}
```

Then you can just add a neat `new()` method for your `Val` type:

```rust
impl Val {
    fn new(n: i32) -> Calculus<Val> {
        Calculus {
            calc: Val { value: n },
        }
    }
}
```

and use the whole thing like this:

```rust
fn main() {
    let a = Val::new(1);
    let b = Val::new(2);
    let c = Val::new(3);

    let e = ((a + b) * c).get_value();

    print!("{}", e);
}
```


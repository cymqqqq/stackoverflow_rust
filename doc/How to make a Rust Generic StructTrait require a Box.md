How to make a Rust Generic Struct/Trait require a Box?

I have a trait `Agent` representing an agent in a simulation, and a struct `SimpleAgent` that implements this trait. Since the size of `Agent` is not known at compile-time, my code generally uses `Vec<Box<dyn Agent>>` I want to create a generic trait `AgentCollection<T>` and implement it with an `AgentTree<T>` struct.

So far I have the following:

```rust
pub trait AgentCollection<T> {
    fn new(agents: Vec<Box<T>>) -> Self;
    fn get_in_rectilinear_range(point: vec::Vec2, range: f64) -> Vec<Box<T>>;
    fn get_in_euclidean_range(point: vec::Vec2, range: f64) -> Vec<Box<T>>;
}

pub struct AgentTree<T: agent::Agent> {
    left: Option<Box<AgentTree<T>>>,
    right: Option<Box<AgentTree<T>>>,
    node: Box<T>,
}

#[allow(unused)]
impl<T: agent::Agent> AgentTree<T> {
    fn range_search(point: vec::Vec2, range: f64) -> std::vec::Vec<Box<T>> {
        todo!()
    }
}

impl<T: agent::Agent> AgentCollection<T> for AgentTree<T> {
    fn new(agents: std::vec::Vec<Box<T>>) -> Self {
        todo!()
    }

    fn get_in_rectilinear_range(point: vec::Vec2, range: f64) -> std::vec::Vec<Box<T>> {
        todo!()
    }

    fn get_in_euclidean_range(point: vec::Vec2, range: f64) -> std::vec::Vec<Box<T>> {
        todo!()
    }
}
```

This all type checks. However, when I go to use it in my main file, e.g.

```rust
let agent_tree = AgentTree::new(last_agents);
```

where `last_agents` has type `std::vec::Vec<std::boxed::Box<dyn agent::Agent>>`, I get the error `the size for values of type 'dyn agent::Agent' cannot be known at compilation time`.

I think that I want to somehow constrain the `AgentTree` type parameter to `Box<agent::Agent` rather than just `agent::Agent`, so that it is sized, but I don't know how to do that. I have tried for example: `pub struct AgentTree<T: Box<agent::Agent>> { ... }`.

answer1

By default, generic type parameters will have a `Sized` trait bound that requires a sized type. But trait objects like `dyn Agent` are not sized, they can represent many types and thus their size cannot be known at compile-time.

To make your code work with `dyn Agent`, all you have to do is to add `?Sized` in your trait bounds to remove the default `Sized` trait bound:

```rust
pub trait AgentCollection<T: ?Sized> {
//                           ^^^^^^
    fn new(agents: Vec<Box<T>>) -> Self;
    fn get_in_rectilinear_range(point: vec::Vec2, range: f64) -> Vec<Box<T>>;
    fn get_in_euclidean_range(point: vec::Vec2, range: f64) -> Vec<Box<T>>;
}

pub struct AgentTree<T: ?Sized + agent::Agent> {
//                      ^^^^^^
    left: Option<Box<AgentTree<T>>>,
    right: Option<Box<AgentTree<T>>>,
    node: Box<T>,
}

#[allow(unused)]
impl<T: ?Sized + agent::Agent> AgentTree<T> {
//      ^^^^^^
    fn range_search(point: vec::Vec2, range: f64) -> std::vec::Vec<Box<T>> {
        todo!()
    }
}

impl<T: ?Sized + agent::Agent> AgentCollection<T> for AgentTree<T> {
//      ^^^^^^
    fn new(agents: std::vec::Vec<Box<T>>) -> Self {
        todo!()
    }

    fn get_in_rectilinear_range(point: vec::Vec2, range: f64) -> std::vec::Vec<Box<T>> {
        todo!()
    }

    fn get_in_euclidean_range(point: vec::Vec2, range: f64) -> std::vec::Vec<Box<T>> {
        todo!()
    }
}
```

answer2

Figured it out, I added the following:

```rust
type BoxedAgent = Box<dyn agent::Agent>;
```

Then used `BoxedAgent` in place of `Box<T: agent::Agent>`.


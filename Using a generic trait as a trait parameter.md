 Using a generic trait as a trait parameter

How can I use related generic types in Rust?

Here's what I've got (only the first line is giving me trouble):

```rust
impl<G, GS> TreeNode<GS> where G: Game, GS: GameState<G>{
    pub fn expand(&mut self, game: &G){
        if !self.expanded{
            let child_states = self.data.generate_children(game);
            for state in child_states{
                self.add_child_with_value(state);
            }
        }
    }
}
```

`GameState` is a trait that is generic to a `Game`, and `self.data` implements `GameState<Game>` of this type. The compiler tells me

```all
error[E0207]: the type parameter `G` is not constrained by the impl trait, self type, or predicates
  --> src/mcts.rs:42:6
   |
42 | impl<G, GS> TreeNode<GS> where G: Game, GS: GameState<G>{
   |      ^ unconstrained type parameter

error: aborting due to previous error
```

but it seems to me like I'm constraining `G` both in the `expand` function, and in the fact that `G` needs to belong to `GS`. I would really appreciate any help.

Edit: Here are some more definitions as of now

```rust
trait GameState<G: Game>: std::marker::Sized + Debug{
    fn generate_children(&self, game: &G) -> Vec<Self>;
    fn get_initial_state(game: &G) -> Self;
}

trait Game{}

struct TreeNode<S> where S: Sized{
    parent: *mut TreeNode<S>,
    expanded: bool,
    pub children: Vec<TreeNode<S>>,
    pub data: S,
    pub n: u32
}

impl<S> TreeNode<S>{
    pub fn new(data: S) -> Self{
        TreeNode {
            parent: null_mut(),
            expanded: false,
            children: vec![],
            data,
            n: 0
        }
    }

    pub fn add_child(&mut self, mut node: TreeNode<S>){
        node.parent = self;
        self.children.push(node);
    }

    pub fn add_child_with_value(&mut self, val: S){
        let new_node = TreeNode::new(val);
        self.add_child(new_node);
    }

    pub fn parent(&self) -> &Self{
        unsafe{
            &*self.parent
        }
    }

}
```

Answer 1

```rust
impl<G, GS> TreeNode<GS> where G: Game, GS: GameState<G>{
    // ...
}
```

The problem is that `G` is not constrained, so there may be multiple (possibly conflicting) implementations in this block, since `GS` maybe implement `GameState<G>` for multiple `G`. The parameter `G` is ambiguous.

------

**If you want to keep `GameState<G>` able to be implemented for multiple `G`**, you should move the constraints from the `impl` block to the method instead:

```rust
// note: G is now a type parameter of the method, not the impl block, which is fine
impl<GS> TreeNode<GS> {
    pub fn expand<G>(&mut self, game: &G) where G: Game, GS: GameState<G> {
        if !self.expanded{
            let child_states = self.data.generate_children(game);
            for state in child_states{
                self.add_child_with_value(state);
            }
        }
    }
}
```

------

**If you only want `GameState` to be implemented for a single `G`**, you should make `G` an associated type of `GameState` instead of a generic type parameter:

```rust
trait GameState: std::marker::Sized + Debug {
    type G: Game;
    fn generate_children(&self, game: &Self::G) -> Vec<Self>;
    fn get_initial_state(game: &Self::G) -> Self;
}

// note: now G is given by the GameState implementation instead of
//       being a free type parameter
impl<GS> TreeNode<GS> where GS: GameState {
    pub fn expand(&mut self, game: &GS::G){
        if !self.expanded{
            let child_states = self.data.generate_children(game);
            for state in child_states{
                self.add_child_with_value(state);
            }
        }
    }
}
```

Answer 2

The concrete type of `G` cannot be detemined based on the type of `TreeNode<GS>`; it is only known when `expand` is called. Note that `expand` could be called twice with *different* types for `G`.

You can express this by constraining the type parameters for the method instead of the entire implementation block:

```rust
impl<GS> TreeNode<GS> {
    pub fn expand<G>(&mut self, game: &G)
    where
        G: Game,
        GS: GameState<G>,
    {
        if !self.expanded {
            let child_states = self.data.generate_children(game);
            for state in child_states {
                self.add_child_with_value(state);
            }
        }
    }
}
```

------

If it should not be possible for `expand` to be called with different `G`s then this is a problem of your modeling. Another way to fix this is to ensure that the type of `G` is known for all `TreeNode`s. e.g.:

```rust
struct TreeNode<G, S>
where
    S: Sized,
{
    parent: *mut TreeNode<G, S>,
    expanded: bool,
    pub children: Vec<TreeNode<G, S>>,
    pub data: S,
    pub n: u32,
}
```

And then your original implementation block should work as written, once you account for the extra type parameter.
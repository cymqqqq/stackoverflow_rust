How to achieve behavior similar to @override in Rust

I'm new to rust and am currently rewriting some of my old java code in it. It's my first time not programming in OOP so this is strange and new to me.

I'm having some problems understanding how to implement a different method (with the same name) to each instance of a struct. In essence I'm trying to achieve behavior similar to **abstract Class**, **extends**, **@override** in Java.

Perhaps this example is better at explaining what exactly I'm trying to do. In it I try to implement different execute() logic to each instance of AbstractNode.

Create a struct called "AbstractNode" that holds some data and has 3 methods associated with it (validate(), log(), execute())

```rust
struct AbstractNode {
    pub data: //data here
    pub validate: bool,
    pub log: String,
} 

trait NodeFunctions {
    fn validate(&self)->bool{false}
    fn log(&self){println!("/")}
    fn execute(&self){}
}

impl NodeFunctions for AbstractNode{
    fn validate(&self)->bool{
        self.validate
    }
    fn log(&self){
        println!("{}/", self.log);
    }
    fn execute(&self){
        //--this function is the problem, because I don't want its behavior to be 
        //shared between all instances of Abstract node--
    }
}
```

I then instantiate several nodes. **If possible I would also like to define the body of the execute() somewhere in here.**

```rust
let node1 = AbstractNode{
    data: //data
    validate: false,
    log: "node1".to_string(),
};

let node2 = AbstractNode{
    data: //data
    validate: 1>0,
    log: "node2".to_string(),
};

let node3 = AbstractNode{
    data: //data
    validate: true,
    log: "node3".to_string(),
};
//...
```

It is called from main like so. If the condition in validate() is true first the log() method is executed, which is the same for all nodes. Then the execute() which is not the same for all nodes.

```rust
fn main(){
    let mut node_tree = vec![
        node1,
        node2,
        node3
        //...
    ];

    for node in node_tree.iter() {
        if node.validate(){
            node.log();
            node.execute(); //<--
            break;
        }
    };
}
```

Each node should be able to hold different logic under the execute() method and I don't know how I could define this specific behavior.

I hope this question is clear enough. If you don't understand what I'm traying to achieve, please ask additional questions.

Ty in advance.

answer1

You could *somewhat* replicate it using closures. However, you'll still end up with parts that *can't* be generic per `Node` if you also want to be able to mutate it within the closure.

*I've renamed and removed some parts, to simplify the examples.*

First, you'll need a `NodeData` type, which holds your data. I'm assuming you want to be able to mutate it within the "`execute`" method. Then you'll need a `Node` type, which holds the data, along with the boxed closure for that `Node` instance.

```rust
struct NodeData {}

struct Node {
    data: NodeData,
    f: Box<dyn Fn(&mut NodeData)>,
}
```

Then we'll implement a method for creating a `Node` instance, along with the `execute` method that calls the closure.

This is where the limitation of using closures appears. You cannot pass it a mutable reference to the `Node` itself. Because the `Node` becomes borrowed when you access `self.f` to call the closure.

```rust
impl Node {
    fn with<F>(f: F) -> Self
    where
        F: Fn(&mut NodeData) + 'static,
    {
        Self {
            data: NodeData {},
            f: Box::new(f),
        }
    }

    fn execute(&mut self) {
        (self.f)(&mut self.data);
    }
}
```

An example of using it would then look like this:

```rust
let mut nodes: Vec<Node> = vec![];

nodes.push(Node::with(|_node_data| {
    println!("I'm a node");
}));
nodes.push(Node::with(|_node_data| {
    println!("I'm another node");
}));
nodes.push(Node::with(|_node_data| {
    println!("I'm also a node");
}));

for node in &mut nodes {
    node.execute();
}
```

Now, again this *works*. But `NodeData` cannot be generic, as then modifying the data in the closure becomes increasingly difficult.

Of course you could defer to having the `NodeData` be a `HashMap`, and that way you can store *anything* with a `String` key and some `enum` value.

------

While you didn't want to have separate types. This does somewhat make it easier, as all node *types* can have different kinds of data.

Because now we can have a single `trait Node` which has the `execute` method.

```rust
trait Node {
    fn execute(&mut self);
}
```

Now define multiple types and implement `Node` for each of them. Again, the two good things of using a trait instead of closure is:

1. Every node you define, can contain any kind of data you'd like
2. In this case `execute` will actually be able to modify `Self`, which the closure solution cannot.

```rust
struct NodeA {}
struct NodeB {}
struct NodeC {}

impl Node for NodeA {
    fn execute(&mut self) {
        println!("I'm a node");
    }
}

impl Node for NodeB {
    fn execute(&mut self) {
        println!("I'm another node");
    }
}

impl Node for NodeC {
    fn execute(&mut self) {
        println!("I'm also a node");
    }
}
```

You can still have a single `Vec` of `nodes` as the traits can easily be boxed.

```rust
let mut nodes: Vec<Box<dyn Node>> = vec![];

nodes.push(Box::new(NodeA {}));
nodes.push(Box::new(NodeB {}));
nodes.push(Box::new(NodeC {}));

for node in &mut nodes {
    node.execute();
}
```

answer2

You could have `AbstractNode` store a closure that takes a reference to `Self`:

```rust
struct AbstractNode {
    pub validate: bool,
    pub log: String,
    pub executor: Box<dyn Fn(&Self)>
}
```

The `NodeFunctions` implementation for `AbstractNode` would simply call the `executor` closure:

```rust
impl NodeFunctions for AbstractNode {
    fn execute(&self) {
        (self.executor)(&self)
    }
    // ...
}
```

Now, everytime you create a new instance of an `AbstractNode`, you can have a custom `executor` function. The `executor` takes a reference to `self` and can therefore access the node's data:

```rust
let node1 = AbstractNode {
  validate: false,
  log: "node1".to_string(),
  executor: Box::new(|n| println!("Executing node #1. Is Valid? {}", n.validate))
};

// => Executing node #1. Is Valid? true
```


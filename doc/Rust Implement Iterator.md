Rust Implement Iterator

So I am currently learning Rust and have a question on how to implement a non-consuming Iterator. I programmed a Stack:

```rust
struct Node<T>{
    data:T,
    next:Option<Box<Node<T>>>
}
pub struct Stack<T>{
    first:Option<Box<Node<T>>>
}
impl<T> Stack<T>{
    pub fn new() -> Self{
        Self{first:None}
    }
    pub fn push(&mut self, element:T){
        let old = self.first.take();
        self.first = Some(Box::new(Node{data:element, next:old}));
    }
    pub fn pop(&mut self) -> Option<T>{
        match self.first.take(){
            None => None,
            Some(node) =>{
                self.first = node.next;
                Some(node.data)
            }
        }
    }
    pub fn iter(self) -> StackIterator<T>{
        StackIterator{
            curr : self.first
        }
    }
}
pub struct StackIterator<T>{
    curr : Option<Box<Node<T>>>
}
impl<T> Iterator for StackIterator<T>{
    type Item = T;
    fn next (&mut self) -> Option<T>{
        match self.curr.take(){
            None => None,
            Some(node) => {
                self.curr = node.next;
                Some(node.data)
            }
        }
    }
}
```

With a Stack Iterator, which is created calling the `iter()` Method on a Stack. The Problem: I had to make this `iter()` method consuming its Stack, therefore a stack is only iteratable once. How can I implement this method without consuming the Stack and without implementing the copy or clone trait?

Thanks for your help, and sorry for the very basic question :)

answer1

> How can I implement this method without consuming the Stack and without implementing the copy or clone trait?

Have StackIterator *borrow* the stack instead, and the iterator return references to the items. Something along the lines of

```rust
impl<T> Stack<T>{
    pub fn iter(&self) -> StackIterator<T>{
        StackIterator{
            curr : &self.first
        }
    }
}
pub struct StackIterator<'stack, T: 'stack>{
    curr : &'stack Option<Box<Node<T>>>
}
impl<'s, T: 's> Iterator for StackIterator<'s, T>{
    type Item = &'s T;
    fn next (&mut self) -> Option<&'s T>{
        match self.curr.as_ref().take() {
            None => None,
            Some(node) => {
                self.curr = &node.next;
                Some(&node.data)
            }
        }
    }
}
```

(I didn't actually test this code so it's possible it doesn't work)

That's essentially what `std::iter::Iter` does (though it's implemented as a *way* lower level).

That said learning Rust by implementing linked lists [probably isn't the best idea in the world](https://rust-unofficial.github.io/too-many-lists/), linked lists are degenerate graphs, and the borrow checker is not on very friendly terms with graphs.


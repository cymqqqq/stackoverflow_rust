Implement a trait for all types implementing a trait

I have this issue:

- multiple structs implementing a trait `Event`
- all can implement `PartialEq` trait the same way

I considered writing this (short version)

```rust
type Data = Vec<u8>;

trait Event {
    fn data(&self) -> &Data;
}

struct NoteOn {
    data: Data,
}
struct NoteOff {
    data: Data,
}
impl Event for NoteOn {
    fn data(&self) -> &Data {
        &self.data
    }
}
impl Event for NoteOff {
    fn data(&self) -> &Data {
        &self.data
    }
}
impl<T: Event> PartialEq for T {
    fn eq(&self, b: &T) -> bool {
        self.data() == b.data()
    }
}

fn main() {
    println!("Hello, world!");
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=181a167f38dd84f83d20466e5fae5790)

This fails to compile:

```none
error[E0119]: conflicting implementations of trait `std::cmp::PartialEq<&_>` for type `&_`:
  --> src/main.rs:23:1
   |
23 | impl<T: Event> PartialEq for T {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: conflicting implementation in crate `core`:
           - impl<A, B> std::cmp::PartialEq<&B> for &A
             where A: std::cmp::PartialEq<B>, A: ?Sized, B: ?Sized;
   = note: downstream crates may implement trait `Event` for type `&_`

error[E0210]: type parameter `T` must be used as the type parameter for some local type (e.g., `MyStruct<T>`)
  --> src/main.rs:23:1
   |
23 | impl<T: Event> PartialEq for T {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ type parameter `T` must be used as the type parameter for some local type
   |
   = note: only traits defined in the current crate can be implemented for a type parameter
```

What is wrong here? Or is there another way to implement generically this `PartialEq` without having to type it once for `NoteOn` and once for `Noteff`?

answer

Here is my best "attempt" at a possible way to do what i wanted, but in a different way following the remarks from @trentcl and @Shepmaster.

I used an Enum as mentioned by @trentctl to be able to switch between the different types, I keep the enum value within a common struct Event so I can easily wrap the different objects and add more code to the Event. Doing this also helps to make an easy Vec type.

**Here i imagine that i only need the Enum, and not Event and an attribute with the enum, I am still learning the variant enum usage**

I also had issues mentioned by @trentcl about the fact i did not own the Vec type, so i wrapped the Vec in a struct Sec instead of simply aliasing the type. This makes my implementation of PartialEq seperated from the type Vec (if i understand, but i m not sure) => the reason i am troubled here is that I thought that using `type A = B;`did create a new type, but the documentation does state it s aliasing (which could make send to why i m not able to implement PartialEq for Vec) **although in the end I imagine i may be very wrong on that too as the fact i created an artificial struct to just wrap 1 attribute also seems counter productive**

=> So to conclude for now thank you everyone, I will keep working on this to see how it could be more efficient, but I guess I was simply using the wrong stuff for the context of Rust, **I am not sure this is a good answer and would appreciate feedbacks or other suggestions**.

https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=d67b7d993fa6b6285962ee58e9b215e5

```rust
type Data = Vec<u8>;

#[derive(Debug)]
enum EventImpl{
    NoteOn(u8),
    NoteOff(u8)
}
#[derive(Debug)]
struct Event{
    data:Data,
    i:EventImpl
}

impl Event{
    fn new(data:Data)->Self{
        Event{
            i: match data[0]{
                0 => EventImpl::NoteOn(data[1]),
                1 => EventImpl::NoteOff(data[1]),
                _ => panic!("unk")
            },
            data:data
        }
    }

    fn data(&self)->&Data{
        &self.data
    }
}
#[derive(Debug)]
struct Seq{
    pub things:Vec<Event>
}

impl PartialEq for Seq{
    fn eq(&self,o:&Self)->bool{
    // i have to learn the iterator implementation    
        let mut r=o.things.len()==self.things.len();
        if ! r{
            return false;
        }
        for i in 0..o.things.len() {
            r = r && o.things[i]==self.things[i];
        }
        r
    }
}
impl PartialEq for Event{
    fn eq(&self,o:&Self)->bool{
    // i have to learn the iterator implementation    
        std::mem::discriminant(&self.i) == std::mem::discriminant(&o.i) && o.data()==self.data()
    }
}


fn main() {
    let mut s:Seq=Seq{things:vec![Event::new(vec![1,2,3])]};
    s.things.push(Event::new(vec![0,1,2]));
    let s2:Seq=Seq{things:vec![Event::new(vec![1,2,3]),Event::new(vec![0,1,2])]};

    println!("Hello, world! {:?} {:?}",s, s.things[1].data());
    println!("S1 == S2 ? {}",s==s2);
}
```


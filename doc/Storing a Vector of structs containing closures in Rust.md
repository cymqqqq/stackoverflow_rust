Storing a Vector of structs containing closures in Rust

In my Rust application, I'd like to store a Vector of structs containing closures which are called later.

So far I have something like this (for a Tetris game):

```rust
pub struct TimestepTimer {
    pub frameForExecution: i32,
    pub func: Box<dyn Fn() -> ()>
}

pub struct Game {
    pub frame: i32,
    pub timers: Vec<TimestepTimer>
}

impl Game {
    fn add_timer (&mut self, framesFromNow: i32, func: Box<dyn Fn() -> ()>) {
        self.timers.push(TimestepTimer {
            frameForExecution: self.frame + framesFromNow,
            func
        });
    }

    /* Example of a function that would use add_timer */
    pub fn begin_gravity(&mut self) {
        self.add_timer(30, Box::new(|| {
            self.move_down();
            // Gravity happens repeatedly
            self.begin_gravity();
        }));
    }

    // Later on would be a function that checks the frame no.
    // and calls the timers' closures which are due
}
```

This doesn't work at the moment due to E0495, saying the reference created by Box::new cannot outlive the begin_gravity function call, but also 'must be valid for the static lifetime'

I'm quite new to Rust, so this might not at all be the idiomatic solution.

Every different solution I try seems to be met by a different compiler error (though I'm aware that's my fault, not the compiler's)

answer1

your closure keeps a reference to `self` (so that it can call `self.move_down()` and `self.begin_gravity()`) and you're trying to store it inside self → that's impossible.

You can get a similar effect if you change your closure to take a `&mut Game` argument and operate on that:

```rust
pub struct TimestepTimer {
    pub frameForExecution: i32,
    pub func: Box<dyn Fn(&mut Game) -> ()>
}

pub struct Game {
    pub frame: i32,
    pub timers: Vec<TimestepTimer>
}

impl Game {
    fn add_timer (&mut self, framesFromNow: i32, func: Box<dyn Fn(&mut Game) -> ()>) {
        self.timers.push(TimestepTimer {
            frameForExecution: self.frame + framesFromNow,
            func
        });
    }

    /* Example of a function that would use add_timer */
    pub fn begin_gravity(&mut self) {
        self.add_timer(30, Box::new(|s: &mut Game| {
            // self.move_down();
            // Gravity happens repeatedly
            s.begin_gravity();
        }));
    }

    // Later on would be a function that checks the frame no.
    // and calls the timers' closures which are due
}
```

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=54f05bb7f9d9f2ab6cb1b7b9b461a6e0)

But now you will get into trouble when calling the timers because you will need a reference to `self.timers` to iterate over it, and at the same time you will need to pass a mutable reference to `self` into the closures, but the compiler won't allow you to have an immutable reference to `self.timers` and a mutable reference to `self` at the same time. If you really want to keep your timers in a `Vec`, the easiest way to solve that will be to swap out `timers` with an empty vector and repopulate as you go:

```rust
pub fn step (&mut self) {
    self.frame += 1;
    let mut timers = vec![];
    std::mem::swap (&mut self.timers, &mut timers);
    for t in timers {
        if self.frame > t.frameForExecution {
            (t.func)(self);
        } else {
            self.timers.push (t);
        }
    }
}
```

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=839aea9f80602108b34de7d358002e69)

However it would probably be better to store the timers in a [`BinaryHeap`](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) which allows a much cleaner and more efficient execution loop:

```rust
pub fn step (&mut self) {
    self.frame += 1;
    while let Some (t) = self.timers.peek() {
        if t.frameForExecution >= self.frame { break; }
        // `unwrap` is ok here because we know from the `peek` that
        // there is at least one timer in `timers`
        let t = self.timers.pop().unwrap();
        (t.func)(self);
    }
}
```

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e5dab16b0d829989530ba152911f439d)

This will require implementing [`Ord`](https://doc.rust-lang.org/std/cmp/trait.Ord.html) along with a few other traits on `TimestepTimer`:

```rust
impl Ord for TimestepTimer {
    fn cmp (&self, other: &Self) -> Ordering {
        // Note the swapped order for `self` and `other` so that the `BinaryHeap`
        // will return the earlier timers first.
        other.frameForExecution.cmp (&self.frameForExecution)
    }
}

impl PartialOrd for TimestepTimer {
    fn partial_cmp (&self, other: &Self) -> Option<Ordering> {
        Some (self.cmp (other))
    }
}

impl PartialEq for TimestepTimer {
    fn eq (&self, other: &Self) -> bool {
        self.frameForExecution == other.frameForExecution
    }
}

impl Eq for TimestepTimer {}
```


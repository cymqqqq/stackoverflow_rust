Deref coercion with generics

I am trying to write a parameterized function `if_found_update` that updates a value in the hash if it exists:

```rust
use std::collections::HashMap;

fn if_found_update<K, V>(data: &mut HashMap<K, V>, k: &K, v: &V, f: &Fn(&V, &V) -> V) -> bool
    where K: std::cmp::Eq,
          K: std::hash::Hash
{
    if let Some(e) = data.get_mut(k) {
        *e = f(e, v);
        return true;
    }
    false
}

fn main() {
    let mut h: HashMap<String, i64> = HashMap::new();
    h.insert("A".to_string(), 0);
    let one = 1 as i64;
    fn update(e1: &i64, e2: &i64) -> i64 {
        e1 + e2
    };
    let k: &str = &"A".to_string();
    println!("{}",
             if_found_update(&mut h, &"A".to_string(), &one, &update)); // works
    println!("{}", if_found_update(&mut h, k, &one, &update)); // fails to compile
}
```

`if_found_update(&mut h, &"A".to_string(), &one, &update);` works fine, but `if_found_update(&mut h, k, &one, &update)` fails to compile with:

```none
error[E0308]: mismatched types
  --> src/main.rs:24:44
   |
24 |     println!("{}", if_found_update(&mut h, k, &one, &update)); // fails to compile
   |                                            ^ expected struct `std::string::String`, found str
   |
   = note: expected type `&std::string::String`
   = note:    found type `&str`
```

I think it's because it fails to for the appropriate deref coercion. Is there a way to get something like this to work?

answer

Some of [`HashMap`](https://doc.rust-lang.org/stable/std/collections/struct.HashMap.html)'s methods, i.e. [`get`](https://doc.rust-lang.org/stable/std/collections/struct.HashMap.html#method.get), [`contains_key`](https://doc.rust-lang.org/stable/std/collections/struct.HashMap.html#method.contains_key), [`get_mut`](https://doc.rust-lang.org/stable/std/collections/struct.HashMap.html#method.get_mut) and [`remove`](https://doc.rust-lang.org/stable/std/collections/struct.HashMap.html#method.remove), can receive a borrowed version of the key type. They do this by using the [`Borrow`](https://doc.rust-lang.org/stable/std/borrow/trait.Borrow.html) trait. They are generic over a type parameter `Q` that can be any type that can represent a borrow of a key. It works this way: when `X` implements `Borrow<Y>`, it means that an `&X` can be borrowed as a `&Y`. For example, [`String` implements `Borrow`](https://doc.rust-lang.org/stable/std/string/struct.String.html#implementations), so a `&String` can be borrowed as a `&str`.

You can take advantage of this by introducing an additional type parameter on your function and adding the correct bounds.

```rust
use std::borrow::Borrow;
use std::collections::HashMap;
use std::hash::Hash;

fn if_found_update<K, V, Q>(data: &mut HashMap<K, V>, k: &Q, v: &V, f: &Fn(&V, &V) -> V) -> bool
    where K: Hash + Eq + Borrow<Q>,
          Q: ?Sized + Hash + Eq
{
    if let Some(e) = data.get_mut(k) {
        *e = f(e, v);
        return true;
    }
    false
} 

fn main() {
    let mut h: HashMap<String, i64> = HashMap::new();
    h.insert("A".to_string(), 0);
    let one = 1 as i64;
    fn update(e1: &i64, e2: &i64) -> i64 { e1 + e2 }
    let k: &str = "A";
    println!("{}", if_found_update(&mut h, &"A".to_string(), &one, &update));
    println!("{}", if_found_update(&mut h, k, &one, &update));
}
```


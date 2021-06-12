How to index a String in Rust

I am attempting to index a string in Rust, but the compiler throws an error. My code (Project Euler problem 4, [playground](https://play.rust-lang.org/?gist=b834e00f32edae1521d1268d1e85f13f&version=stable&backtrace=0)):

```rust
fn is_palindrome(num: u64) -> bool {
    let num_string = num.to_string();
    let num_length = num_string.len();

    for i in 0 .. num_length / 2 {
        if num_string[i] != num_string[(num_length - 1) - i] {
            return false;
        }
    }

    true
}
```

The error:

```none
error[E0277]: the trait bound `std::string::String: std::ops::Index<usize>` is not satisfied
 --> <anon>:7:12
  |
7 |         if num_string[i] != num_string[(num_length - 1) - i] {
  |            ^^^^^^^^^^^^^
  |
  = note: the type `std::string::String` cannot be indexed by `usize`
```

Is there a reason why `String` can not indexed? How can I access the data then?

answer1

indexing into a string is not available in Rust. The reason for this is that Rust strings are encoded in UTF-8 internally, so the concept of indexing itself would be ambiguous, and people would misuse it: byte indexing is fast, but almost always incorrect (when your text contains non-ASCII symbols, byte indexing may leave you inside a character, which is really bad if you need text processing), while char indexing is not free because UTF-8 is a variable-length encoding, so you have to traverse the entire string to find the required code point.

If you are certain that your strings contain ASCII characters only, you can use the `as_bytes()` method on `&str` which returns a byte slice, and then index into this slice:

```rust
let num_string = num.to_string();

// ...

let b: u8 = num_string.as_bytes()[i];
let c: char = b as char;  // if you need to get the character as a unicode code point
```

If you do need to index code points, you have to use the `char()` iterator:

```rust
num_string.chars().nth(i).unwrap()
```

As I said above, this would require traversing the entire iterator up to the `i`th code element.

Finally, in many cases of text processing, it is actually necessary to work with [grapheme clusters](https://www.unicode.org/reports/tr29/) rather than with code points or bytes. With the help of the [unicode-segmentation](https://crates.io/crates/unicode-segmentation) crate, you can index into grapheme clusters as well:

```rust
use unicode_segmentation::UnicodeSegmentation

let string: String = ...;
UnicodeSegmentation::graphemes(&string, true).nth(i).unwrap()
```

Naturally, grapheme cluster indexing has the same requirement of traversing the entire string as indexing into code points.

answer2

The correct approach to doing this sort of thing in Rust is not indexing but *iteration*. The main problem here is that Rust's strings are encoded in UTF-8, a variable-length encoding for Unicode characters. Being variable in length, the memory position of the nth character can't determined without looking at the string. This also means that accessing the nth character has a runtime of O(n)!

In this special case, you can iterate over the bytes, because your string is known to only contain the characters 0–9 (iterating over the characters is the more general solution but is a little less efficient).

Here is some idiomatic code to achieve this ([playground](https://play.rust-lang.org/?gist=c121a769db4ecb107a7f74c3f17ae047&version=stable&backtrace=0)):

```rust
fn is_palindrome(num: u64) -> bool {
    let num_string = num.to_string();
    let half = num_string.len() / 2;

    num_string.bytes().take(half).eq(num_string.bytes().rev().take(half))
}
```

We go through the bytes in the string both forwards (`num_string.bytes().take(half)`) and backwards (`num_string.bytes().rev().take(half)`) simultaneously; the `.take(half)` part is there to halve the amount of work done. We then simply compare one iterator to the other one to ensure at each step that the nth and nth last bytes are equivalent; if they are, it returns true; if not, false.

notes:

`String` has a direct [`as_bytes`](http://doc.rust-lang.org/master/collections/string/struct.String.html#method.as_bytes). Furthermore, you can use [`std::iter::order::equals`](http://doc.rust-lang.org/master/std/iter/order/fn.equals.html). rather than the `all`: `equals(iter.take(n), iter.rev().take(n))`.

convention implies importing `std::iter::order` and calling `order::equals(..., ...)`

answer3

If what you are looking for is something similar to an index, you can use

[`.chars()`](https://doc.rust-lang.org/std/string/struct.String.html#method.chars) and [`.nth()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.nth) on a string.

------

`.chars()` -> Returns an iterator over the `char`s of a string slice.

`.nth()` -> Returns the nth element of the iterator, in an [`Option`](https://doc.rust-lang.org/std/option/enum.Option.html)

------

Now you can use the above in several ways, for example:

```rust
let s: String = String::from("abc");
//If you are sure
println!("{}", s.chars().nth(x).unwrap());
//or if not
println!("{}", s.chars().nth(x).expect("message"));
```

notes:

It is important to note that `Chars::nth(n)` *consumes* n characters, rather than just being plain indexing. As stated by the documentation *calling nth(0) multiple times on the same iterator will return different elements*

answer4

You can convert a `String` or `&str` to a `vec` of a chars and then index that `vec`.

For example:

```rust
fn main() {
    let s = "Hello world!";
    let my_vec: Vec<char> = s.chars().collect();
    println!("my_vec[0]: {}", my_vec[0]);
    println!("my_vec[1]: {}", my_vec[1]);
}
```

Here you have a live [example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=d4d6fe3d15a087ee1e92d4d3c227d0de)
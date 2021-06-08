Trying to convert this for loop from c++ to rust and i'm having a hard time figuring it out as I'm very new to Rust syntax.

```cpp
double sinError = 0;
for (float x = -10 * M_PI; x < 10 * M_PI; x += M_PI / 300) {
    double approxResult = sin_approx(x);
    double libmResult = sinf(x);
    sinError = MAX(sinError, fabs(approxResult - libmResult));
}
```

# Iterate over integers

it's usually better to iterate over integers instead of floats, to prevent numerical errors from adding up:

```rust
use std::f32::consts::PI;
let mut sin_error = 0.0;
for x in (-3000..3000).map(|i| (i as f32) * PI / 300.0) {
	sin_error = todo!();
}
```

`Just replace `todo!()` with the code that computes the next `sin_error`.

### A more functional way

```rust
use std::f32::consts::PI;
let sin_error = (-3000..3000).map(|i| (i as f32) * PI / 300.0)
.fold(0.0, |sin_error, x| todo!());
```

In case you don't care about numerical errors, or want to iterate over something else, here are some other options:

# Use a `while` loop

It's not as nice, but does the job!

```rust
use std::f32::consts::PI;
let mut sin_error = 0.0;
let mut x = -10.0 * PI;
while x < 10.0 * PI {
    sin_error = todo!();
    x += PI / 300.0;
}
```

# Create your iterator with `successors()`

The `successors()` function creates a new iterator where each successive item is computed based on the preceding one:

```rust
use std::f32::consts::PI;
use std::iter::successors;
let mut sin_error = 0.0;
let iter = successors(Some(-10.0 * PI), |x| Some(x + PI / 300.0));
for x in iter.take_while(|&x| x < 10.0 * PI) {
    sin_error = todo!();
}
```

### A more functional way

```rust
use std::f32::consts::PI;
use std::iter::successors;
let sin_error = successors(Some(-10.0 * PI), |x| Some(x + PI / 300.0))
.take_while(|&x| x < 10.0 * PI)
.fold(0.0, |sin_error, x| todo!());
```


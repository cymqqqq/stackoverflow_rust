Pass array from C to Rust with dimensions defined during runtime

I am trying to pass a matrix from R to Rust by way of C. I can pass a two-dimensional array if I hard-code the array-dimensions in the rust function signature. Is there a way to do this dynamically by passing a pointer to the array along with the number of rows and columns?

My C code:

```cpp
#include "Rinternals.h"
#include "R.h"
#include <stdint.h>

void test1(double* matrix);

void test2(double* matrix, int32_t nrow, int32_t ncol);

SEXP pass_matrix_to_rust(SEXP mat, SEXP nrow, SEXP ncol) {

    // store nrows and ncols into integers
    int32_t rows = *INTEGER(nrow);
    int32_t cols = *INTEGER(ncol);

    // store pointer to matrix of doubles
    double *matrix = REAL(mat);

    test1(matrix); // hard coded version
    test2(matrix, rows, cols);

    return R_NilValue;
}
```

My Rust code:

```rust
// This function works but I have to specify the size at compile time
#[no_mangle]
pub extern fn test1(value: *const [[f64; 10]; 10]) {
  let matrix = unsafe{*value};

  println!("{:?}", matrix);
}

// this function doesn't compile
#[no_mangle]
pub extern fn test2(value: *const f64, nrow: i32, ncol: i32) {
  let matrix: [[f64; nrow]; ncol] = unsafe{*value};

  println!("{:?}", matrix);
}

// rustc output:
 rustc glue.rs
glue.rs:30:29: 30:33 error: no type for local variable 161
glue.rs:30   let matrix: [[f64; nrow]; ncol] = unsafe{*value};
                                       ^~~~
glue.rs:30:22: 30:26 error: no type for local variable 158
glue.rs:30   let matrix: [[f64; nrow]; ncol] = unsafe{*value};
                                ^~~~
```

answer


Here's what I'd suggest, assuming this is in row-major order.

C code:

```c
void test2(double* matrix, int32_t nrow, int32_t ncol);
```

Rust code:

```rust
use std::slice;

#[no_mangle]
pub extern fn test2(pointer: *const f64, nrow: i32, ncol: i32) {
    let mut rows: Vec<&[f64]> = Vec::new();
    for i in 0..nrow as usize {
        rows.push(unsafe {
            slice::from_raw_parts(
                pointer.offset(i as isize * ncol as isize), 
                ncol as usize
            )
        });
    }
    let matrix: &[&[f64]] = &rows[..];

    println!("{:#?}", matrix);
}
```


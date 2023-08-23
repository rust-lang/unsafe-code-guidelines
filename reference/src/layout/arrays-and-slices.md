# Layout of Rust array types and slices

## Layout of Rust array types 

Array types, `[T; N]`, store `N` values of type `T` with a _stride_ that is
equal to the size of `T`. Here, _stride_ is the distance between each pair of
consecutive values within the array.

The _offset_ of the first array element is `0`, that is, a pointer to the array
and a pointer to its first element both point to the same memory address.

The _alignment_ of array types is greater or equal to the alignment of its
element type. If the element type is `repr(C)` the layout of the array is
guaranteed to be the same as the layout of a C array with the same element type.

> **Note**: the type of array arguments in C function signatures, e.g., `void
> foo(T x[N])`, decays to a pointer. That is, these functions do not take arrays
> as an arguments, they take a pointer to the first element of the array
> instead. Array types are therefore _improper C types_ (not C FFI safe) in Rust
> foreign function declarations, e.g., `extern { fn foo(x: [T; N]) -> [U; M];
> }`. Pointers to arrays are fine: `extern { fn foo(x: *const [T; N]) -> *const
> [U; M]; }`, and `struct`s and `union`s containing arrays are also fine.

### Arrays of zero-size

Arrays `[T; N]` have zero size if and only if their count `N` is zero or their
element type `T` is zero-sized.

### Layout compatibility with packed SIMD vectors

The [layout of packed SIMD vector types][Vector] [^2] requires the _size_ and
_alignment_ of the vector elements to match. That is, types with [packed SIMD
vector][Vector] layout are layout compatible with arrays having the same element
type and the same number of elements as the vector.

[^2]: The [packed SIMD vector][Vector] layout is the layout of `repr(simd)` types like [`__m128`].

[Vector]: packed-simd-vectors.md
[`__m128`]: https://doc.rust-lang.org/core/arch/x86_64/struct.__m128.html

## Layout of Rust slices

The layout of a slice `[T]` of length `N` is the same as that of a `[T; N]` array. 

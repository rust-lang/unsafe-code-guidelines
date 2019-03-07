# Layout of Rust array types and slices

## Layout of Rust array types 

Array types, `[T; N]`, store `N` values of type `T` with a constant _stride_.
Here, _stride_ is the distance between each pair of consecutive values within
the array.

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

The _stride_ of the array is constant for all element pairs and it is computed
as the _size_ of the element type rounded up to the next multiple of the
_alignment_ of the element type.

### Special case `stride == size`

When the element _size_ is a multiple of the element's _alignment_, then `stride
== size`, and the elements are laid out contiguously in memory, e.g., `[u8; 4]`.
In this case, the _size_ of the array can be computed as `size_of::<T>() * N`,
and a pointer to the `i`-th element of the array can be obtained by offsetting a
pointer to the first element of the array by `i`[^1].

> **Note:** In the current Rust implementation, _size_ is always a multiple of
> the element's _alignment_, and therefore `stride == size` always holds. This
> is, however, not guaranteed by the [layout of structs and tuples].

[^1]: When `stride > size` the pointer needs to be advanced by the array
    _stride_ instead of by the element _size_.

[layout of structs and tuples]: ./structs-and-tuples.md

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

## Unresolved questions

### Guaranteeing `stride == size` ?

Currently, the [layout of structs and tuples] does not guarantee that the
element _size_ is a multiple of its _alignment_. For example, consider:

```rust,ignore
struct A(u16, u8);
type B = [A; 4];
```

In the current Rust implementation, `A` has an alignment and a size of `4`, and
`B` has a size of `16`, such that `B` contains four `A`s that are contiguously
laid in memory. 

However, a future Rust implementation could implement a layout optimization that
reduces the size of `A` to `3`. For the elements of `B` to be properly aligned,
`B` would need to choose a `stride == 4`, resulting in a `stride > size`.

Guaranteeing `stride >= size` is forward-compatible with such
layout-optimization proposals:
  
  * [rust-lang/rfcs/1397: Spearate size and stride for types](https://github.com/rust-lang/rfcs/issues/1397)
  * [rust-lang/rust/17027: Collapse trailing padding](https://github.com/rust-lang/rust/issues/17027)

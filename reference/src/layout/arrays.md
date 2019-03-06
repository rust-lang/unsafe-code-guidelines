# Layout of Rust array types

Array types, `[T; N]`, store `N` values of type `T` contiguously with a constant
_stride_ (the distance between each two consecutive values).

The offset of the first array element is `0`.

The stride of the array is computed as the size of the element type rounded up
to the next multiple of the alignment of the element type. That is, the _stride_
of an array can be larger than the element size iff the alignment requirements
of the element are larger than its size.

Having a stride larger than the element size allows:

```rust,ignore
struct A(u16, u8); // size_of == 3, align_of == 4
type B = [A; 4];  // => stride == 4 > 3
```

The size and alignment of the element types supported by the [Vector] layout
[^1] match. Therefore, types with [Vector] layout are layout compatible with
arrays of the same element type and number of elements.

[^1]: The Vector layout is the layout of `repr(simd)` types like `__m128`.
[Vector]: ./vectors.md

## Unresolved questions

### Stride > size

In the current Rust implementation, `size >= align` and hence `stride == size`
always holds. However, `stride == size` is not guaranteed here to be
forward-compatible with proposals allowing `stride > size`, like:
  
  * [rust-lang/rfcs/1397: Spearate size and stride for types](https://github.com/rust-lang/rfcs/issues/1397)
  * [rust-lang/rust/17027: Collapse trailing padding](https://github.com/rust-lang/rust/issues/17027)

# Layout of Rust array types

Array types, `[T; N]`, store `N` values of type `T` with a constant
_stride_, where _stride_ is the distance between each pair of consecutive values
within the array.

The _offset_ of the first array element is `0`, that is, a pointer to the array
and a pointer to its first element have the same address.

The _stride_ of the array is computed as the _size_ of the element type rounded up
to the next multiple of the _alignment_ of the element type.

When the element _size_ is a multiple of the element's _alignment_, then `stride
== size`, and the elements are laid out contiguously in memory, e.g., `[u8; 4]`.
In this case, the size of the array can be computed as `size_of::<T>() * N`[^1].

[^1]: The alignment of the array is, however, unspecified.

The [layout of Vector types] [^2] requires the size and alignment of the Vector
elements to match. That is, types with Vector layout are layout compatible with
arrays having the same element type and number of elements as the Vector.

[^2]: The Vector layout is the layout of `repr(simd)` types like `__m128`.
[Vector]: ./vectors.md

## Unresolved questions

### Guaranteeing `stride == size` ?

The element _size_ is not guaranteed to be a multiple of the element's
alignment. In:

```rust,ignore
struct A(u16, u8);
type B = [A; 4];
```

and in the current Rust implementation, `A` has an alignment and a size of `4`,
and `B` has a size of `16`. However, the size of `A` is not guaranteed by the
[layout of structs and tuples].

[layout of structs and tuples]: ./structs-and-tuples.md

If a future Rust implementation wanted to use a smaller size for `A`, that is, a
`size == 3`, then `B` would need a `stride == 4` to ensure that its elements are
properly aligned, resulting in a `stride > size`.

The current guarantee that `stride >= size` is forward-compatible with the open
proposals about allowing `A` to have a size of `3`:
  
  * [rust-lang/rfcs/1397: Spearate size and stride for types](https://github.com/rust-lang/rfcs/issues/1397)
  * [rust-lang/rust/17027: Collapse trailing padding](https://github.com/rust-lang/rust/issues/17027)

# Layout of Rust array types

Array types, `[T; N]`, store `N` values of type `T` with a constant
_stride_, where _stride_ is the distance between each pair of consecutive values
within the array.

The _offset_ of the first array element is `0`, that is, a pointer to the array
and a pointer to its first element point to the same memory address.

The _stride_ of the array is computed as the _size_ of the element type rounded up
to the next multiple of the _alignment_ of the element type.

When the element _size_ is a multiple of the element's _alignment_, then `stride
== size`, and the elements are laid out contiguously in memory, e.g., `[u8; 4]`.
In this case, the size of the array can be computed as `size_of::<T>() * N`[^1].

> **Note:** In the current Rust implementation, _size_ is always a multiple of
> the element's _alignment_, and therefore `stride == size` always holds. This
> is, however, not guaranteed by the [layout of structs and tuples].

[^1]: The alignment of the array is, however, unspecified. For example, the
[SysV AMD64 ABI] requires array arguments to be at least 16 byte aligned to
allow the use of SSE instructions.

[layout of structs and tuples]: ./structs-and-tuples.md
[SysV AMD64 ABI]: https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf

The [layout of Vector types] [^2] requires the _size_ and _alignment_ of the Vector
elements to match. That is, types with Vector layout are layout compatible with
arrays having the same element type and number of elements as the Vector.

[^2]: The Vector layout is the layout of `repr(simd)` types like `__m128`.
[Vector]: ./vectors.md

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

However, a future Rust implementation could, as a layout optimization, choose a
smaller size for `A` (that is, `3`). For `A` elements to be properly aligned
within `B`, then `B` would need to choose a `stride == 4`, resulting in a
`stride > size`.

Guaranteeing `stride >= size` is forward-compatible such the layout-optimization
proposals:
  
  * [rust-lang/rfcs/1397: Spearate size and stride for types](https://github.com/rust-lang/rfcs/issues/1397)
  * [rust-lang/rust/17027: Collapse trailing padding](https://github.com/rust-lang/rust/issues/17027)

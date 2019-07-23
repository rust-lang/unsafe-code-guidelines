# Layout of reference and pointer types

### Terminology

Reference types are types of the form `&T`, `&mut T`.

Raw pointer types are types of the form `*const T` or `*mut T`.

### Representation

The alignment of `&T`, `&mut T`, `*const T` and `*mut T` are the same,
and are at least the word size.

* If `T` is a sized type then the alignment of `&T` is the word size.
* The alignment of `&dyn Trait` is the word size.
* The alignment of `&[T]` is the word size.
* The alignment of `&str` is the word size.
* Alignment in other cases may be more than the word size (e.g., for other dynamically sized types).

The sizes of `&T`, `&mut T`, `*const T` and `*mut T` are the same,
and are at least one word.

* If `T` is a sized type then the size of `&T` is one word.
* The size of `&dyn Trait` is two words.
* The size of `&[T]` is two words.
* The size of `&str` is two words.
* Size in other cases may be more than one word (e.g., for other dynamically sized types).

### Notes

The size, alignment, and call ABI of `&T`, `&mut T`, `*const T` and `*mut T` are
the same.

If `T` is sized, the size, alignment, and call ABI of references and pointers to
`T` is the same as that of C pointers.

> **warning**: while the size, alignment, and call ABI of references and
> pointers is compatible with the layout of C pointers, references come with a
> _validity_ invariant that does not allow them to be used when they could be
> `NULL`, unaligned, dangling, or, in the case of `&mut T`, aliasing. Since the
> niches of references and pointers are not the same, these types are not
> layout-compatible with each other.

We do not make any guarantees about the layout of
multi-trait objects `&(dyn Trait1 + Trait2)` or references to other dynamically sized types,
other than that they are at least word-aligned, and have size at least one word.

The layout of `&dyn Trait` when `Trait` is a trait is the same as that of:
```rust
#[repr(C)]
struct DynObject {
  data: *const u8,
  vtable: *const u8,
}
```

> **note**: In the layout of `&dyn mut Trait` the field `data` is of the type `*mut u8`.

The size, alignment, and call ABI of `&[T]` and `&mut [T]` are the same as that of:
```rust
#[repr(C)]
struct Slice<T> {
  ptr: *const T,
  len: usize,
}
```

> **note**: the field `ptr` is of the type `*mut T` for `&mut [T]`.

The size, alignment, and call ABI of `&str` is the same as that of `&[u8]`, and
the layout of `&mut str` is the same as that of `&mut [u8]`.

#### Unresolved question

* Does `&str` have the same niches as `&[u8]` ?

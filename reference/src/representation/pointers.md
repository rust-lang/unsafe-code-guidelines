# Representation of reference and pointer types

### Terminology

Reference types are types of the form `&T`, `&mut T` or `&dyn T`.

Raw pointer types are types of the form `*const T` or `*mut T`.

### Representation

The alignment of `&T`, `&mut T`, `*const T` and `*mut T` are the same,
and are at least the word size.

* If `T` is a trait, then the alignment of `&dyn T` is the word size.
* If `T` is a sized type then the alignment of `&T` is the word size.
* The alignment of `&[T]` is the word size.
* The alignment of `&str` is the word size.
* Alignment in other cases may be more than the word size (e.g., for other dynamically sized types).

The sizes of `&T`, `&mut T`, `*const T` and `*mut T` are the same,
and are at least one word.

* If `T` is a trait, then the size of `&dyn T` is two words.
* If `T` is a sized type then the size of `&T` is one word.
* The size of `&[T]` is two words.
* The size of `&str` is two words.
* Size in other cases may be more than one word (e.g., for other dynamically sized types).

### Notes

The representations of `&T` and `&mut T` are the same.

We do not make any guarantees about the representation of
multi-trait objects `&(dyn T + U)` or references to other dynamically sized types,
other than that they are at least word-aligned, and have size at least one word.

The representation of `&dyn T` when `T` is a trait is the same as that of:
```rust
#[repr(C)]
struct DynObject {
  data: &u8,
  vtable: &u8,
}
```

The representation of `&[T]` is the same as that of:
```rust
#[repr(C)]
struct Slice<T> {
  ptr: &T,
  len: usize,
}
```

The representation of `&str` is the same as that of `&[u8]`.

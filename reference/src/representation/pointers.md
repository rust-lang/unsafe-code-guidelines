# Representation of reference and pointer types

### Terminology

Reference types are types of the form `&T` or `&mut T`.

Raw pointer types are types of the form `*const T` or `*mut T`.

### Representation

The alignment of reference and raw pointer types is the word size.

The sizes of `&T`, `&mut T`, `*const T` and `*mut T` are the same,
and are at least one word.

* If `T` is a trait, then the size of `&T` is two words.
* If `T` is a sized type then the size of `&T` is one word.
* The size of `&[T]` is two words.

Note that we do not make any guarantees about the sizes of
multi-trait objects `&(dyn T + U)` or references to other dynamically sized types.

### Notes

The representations of `&T` and `&mut T` are the same.

The representation of `&T` when `T` is a trait is the same as that of:
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

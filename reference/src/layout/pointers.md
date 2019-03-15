# Layout of reference and pointer types

### Terminology

Reference types are types of the form `&T`, `&mut T` or `&dyn T`.

Raw pointer types are types of the form `*const T` or `*mut T`.
We write `*T` when the mutability attribute is unimportant.

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

The layouts of `&T`, `&mut T` and `*T` are the same.

If `T` is sized, references and pointers to `T` have a size and alignment of one
word and have therefore the same layout as C pointers.

> **warning**: while the layout of references and pointers is compatible with
> the layout of C pointers, references come with a _validity_ invariant that
> does not allow them to be used when they could be `NULL`, unaligned, dangling,
> or, in the case of `&mut T`, aliasing.

We do not make any guarantees about the layout of
multi-trait objects `&(dyn T + U)` or references to other dynamically sized types,
other than that they are at least word-aligned, and have size at least one word.

The layout of `&dyn T` when `T` is a trait is the same as that of:
```rust,ignore
#[repr(C)]
struct DynObject {
  data: *u8,
  vtable: *u8,
}
```

The layout of `&[T]` is the same as that of:
```rust,ignore
#[repr(C)]
struct Slice<T> {
  ptr: *T,
  len: usize,
}
```

The layout of `&str` is the same as that of `&[u8]`.

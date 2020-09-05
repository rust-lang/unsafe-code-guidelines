# Layout of reference and pointer types

**Disclaimer:** Everything this section says about pointers to dynamically sized
types represents the consensus from issue [#16], but has not been stabilized
through an RFC. As such, this is preliminary information.

[#16]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/16

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

The layouts of `&T`, `&mut T`, `*const T` and `*mut T` are the same.

If `T` is sized, references and pointers to `T` have a size and alignment of one
word and have therefore the same layout as C pointers.

> **warning**: while the layout of references and pointers is compatible with
> the layout of C pointers, references come with a _validity_ invariant that
> does not allow them to be used when they could be `NULL`, unaligned, dangling,
> or, in the case of `&mut T`, aliasing.

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

The layout of `&[T]` is the same as that of:
```rust
#[repr(C)]
struct Slice<T> {
  ptr: *const T,
  len: usize,
}
```

> **note**: In the layout of `&mut [T]` the field `ptr` is of the type `*mut T`.

The layout of `&str` is the same as that of `&[u8]`, and the layout of `&mut str` is
the same as that of `&mut [u8]`.

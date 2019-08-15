# Layout of unions

**Disclaimer:** This chapter represents the consensus from issue
[#13].  The statements in here are not (yet) "guaranteed"
not to change until an RFC ratifies them.

[#13]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/13

### Layout of individual union fields

When laying out an `union`, the compiler has to decide how the fields of the
union are arranged. Union fields are laid out by the compiler "on top" of each
other, that is, the bytes of one field might overlap with the bytes of another
field - as opposed to, e.g., `struct`s, where the fields are laid out "next to"
each other, such that the bytes of one field never overlap with the bytes of
another field. This can be visualized as follows:

```rust,ignore
[ P P [field0_ty] P P P P ]
[ P P P P [field1_ty] P P ]
[ P P P [field2_ty] P P P ]
```

> **Figure: union field layout**: Each row in the picture shows the layout of
> the union for each of its fields, where the square brackets `[]` depict an
> array of bytes. Here, `P` is a byte of type `Pad` and `[field{i}_ty]` is the
> bytes of the type of the `i`-th union field.

The individual fields (`[field{i}_ty_]`) are blocks of fixed size determined by
the field's layout. The compiler picks the offset of the fields with respect to
the union and the `union` size according to certain constraints like, for
example, the alignment requirements of the fields, the `#[repr]` attribute of
the `union`, etc.

### Unions with default layout ("`repr(Rust)`")

The default layout of Rust unions is **unspecified**. 

That is, there are no guarantees about the offset of the fields, whether all
fields have the same offset, etc.

<details><summary><b>Rationale</b></summary>

As of this writing, we want to keep the option of using non-zero offsets open
for the future; whether this is useful depends on what exactly the
compiler-assumed invariants about union contents are. This might become clearer
after the validity of unions
([unsafe-code-guidelines/73](https://github.com/rust-lang/unsafe-code-guidelines/issues/73))
is settled.

Even if the offsets happen to be all 0, there might still be differences in the
function call ABI.  If you need to pass unions by-value across an FFI boundary,
you have to use `#[repr(C)]`.

</details>

#### Layout of unions with a single non-zero-sized field

The layout of unions with a single non-zero-sized field is the same as the
layout of that field if:

* that field has no padding bits, and
* the alignment requirement of all zero-sized fields is 1.

For example, here:

```rust
# use std::mem::{size_of, align_of};
# #[derive(Copy, Clone)]
#[repr(transparent)]
struct SomeStruct(i32);
# #[derive(Copy, Clone)]
struct Zst;
union U0 {
   f0: SomeStruct,
   f1: Zst,
}
# fn main() {
# assert_eq!(size_of::<U0>(), size_of::<SomeStruct>());
# assert_eq!(align_of::<U0>(), align_of::<SomeStruct>());
# }
```

the union `U0` has the same layout as `SomeStruct`, because `SomeStruct` has no
padding bits - it is equivalent to an `i32` due to `repr(transparent)` - and
because the alignment of `Zst` is 1.

On the other hand, here:

```rust
# use std::mem::{size_of, align_of};
# #[derive(Copy, Clone)]
struct SomeOtherStruct(i32);
# #[derive(Copy, Clone)]
#[repr(align(16))] struct Zst2;
union U1 {
   f0: SomeOtherStruct,
   f1: Zst2,
}
# fn main() {
# assert_eq!(size_of::<U1>(), align_of::<Zst2>());
# assert_eq!(align_of::<U1>(), align_of::<Zst2>());
assert_eq!(align_of::<Zst2>(), 16);
# }
```

the alignment requirement of `Zst2` is not 1, and `SomeOtherStruct` has an
unspecified layout and could contain padding bits. Therefore, the layout of `U1`
is **unspecified**.

### C-compatible layout ("repr C")

The layout of `repr(C)` unions follows the C layout scheme. Per sections
[6.5.8.5] and [6.7.2.1.16] of the C11 specification, this means that the offset
of every field is 0. Unsafe code can cast a pointer to the union to a field type
to obtain a pointer to any field, and vice versa. 

[6.5.8.5]: http://port70.net/~nsz/c/c11/n1570.html#6.5.8p5
[6.7.2.1.16]: http://port70.net/~nsz/c/c11/n1570.html#6.7.2.1p16

#### Padding

Since all fields are at offset 0, `repr(C)` unions do not have padding before
their fields. They can, however, have padding in each union variant *after* the
field, to make all variants have the same size.

Moreover, the entire union can have trailing padding, to make sure the size is a
multiple of the alignment:

```rust
# use std::mem::{size_of, align_of};
#[repr(C, align(2))]
union U { x: u8 }
# fn main() {
// The repr(align) attribute raises the alignment requirement of U to 2
assert_eq!(align_of::<U>(), 2);
// This introduces trailing padding, raising the union size to 2
assert_eq!(size_of::<U>(), 2);
# }
```

> **Note**: Fields are overlapped instead of laid out sequentially, so 
> unlike structs there is no "between the fields" that could be filled 
> with padding.

#### Zero-sized fields

`repr(C)` union fields of zero-size are handled in the same way as in struct
fields, matching the behavior of GCC and Clang for unions in C when zero-sized
types are allowed via their language extensions.

That is, these fields occupy zero-size and participate in the layout computation
of the union as usual:

```rust
# use std::mem::{size_of, align_of};
#[repr(C)] 
union U {
  x: u8,
  y: [u16; 0],
}
# fn main() {
// The zero-sized type [u16; 0] raises the alignment requirement to 2
assert_eq!(align_of::<U>(), 2);
// This in turn introduces trailing padding, raising the union size to 2
assert_eq!(size_of::<U>(), 2);
# }
```

**C++ compatibility hazard**: C++ does, in general, give a size of 1 to types
with no fields. When such types are used as an union field in C++, a "naive"
translation of that code into Rust will not produce a compatible result. Refer
to the [struct chapter](structs-and-tuples.md#c-compatible-layout-repr-c) for
further details.

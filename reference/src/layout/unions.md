# Layout of unions

**Disclaimer**: This chapter is a work-in-progress.
What's contained here represents the consensus from [various issues][union
discussion].
The statements in here are not (yet) "guaranteed" not to change until an RFC
ratifies them.

[union discussion]: https://github.com/rust-lang/unsafe-code-guidelines/blob/master/active_discussion/unions.md

### Layout of individual union fields

A union consists of several variants, one for each field. All variants have the
same size and start at the same memory address, such that in memory the variants
overlap. This can be visualized as follows:

```text
[ <--> [field0_ty] <----> ]
[ <----> [field1_ty] <--> ]
[ <---> [field2_ty] <---> ]
```
**Figure 1** (union-field layout): Each row in the picture shows the layout of
the union for each of its variants. The `<-...->` and `[ ... ]` denote the
differently-sized gaps and fields, respectively.

The individual fields (`[field{i}_ty_]`) are blocks of fixed size determined by
the field's [layout]. Since we allow creating references to union fields
(`&u.i`), the only degrees of freedom the compiler has when computing the layout
of a union are the size of the union, which can be larger than the size of its
largest field, and the offset of each union field within its variant. How these
are picked depends on certain constraints like, for example, the alignment
requirements of the fields, the `#[repr]` attribute of the `union`, etc.

Unions may contain both [exterior][exterior padding] and [interior padding].
In the below diagram, exterior padding is marked by `EXT`, interior padding by
`INT`, and bytes that are padding bytes for a particular field but not padding
for union as a whole are marked `NON`:

```text
[ EXT [ field0_0_ty | INT | field0_1_ty | INT ] EXT ]
[ EXT [ field1_0_ty | INT | NON NON NON | INT ] EXT ]
[ EXT | NON NON NON | INT [ field2_0_ty ] INT | EXT ]
```

It is necessarily the case that any byte that is a non-padding byte for any
field is also a non-padding byte for the union.
It is, in general, **unspecified** whether the converse is true.
Specific reprs may specify whether or not bytes are padding bytes.

Padding bytes in unions has subtle implications; see the union [value model].

### Unions with default layout ("`repr(Rust)`")

Except for the guarantees provided below for some specific cases, the default
layout of Rust unions is, _in general_, **unspecified**.

That is, there are no _general_ guarantees about the offset of the fields,
whether all fields have the same offset, what the call ABI of the union is, etc.

**Major footgun:** The layout of `#[repr(Rust)]` enums allows for the [padding footgun] to also exist with `#[repr(Rust)]`, and this behaviour *is*
extant in Rustc as of this writing. It is [**TBD**][#354] whether it will be
removed.

[padding footgun]: #padding-footgun

<details><summary><b>Rationale</b></summary>

As of this writing, we want to keep the option of using non-zero offsets open
for the future; whether this is useful depends on what exactly the
compiler-assumed invariants about union contents are. This might become clearer
after the [validity of unions][#73] is settled.

Even if the offsets happen to be all 0, there might still be differences in the
function call ABI.  If you need to pass unions by-value across an FFI boundary,
you have to use `#[repr(C)]`.

[#73]: https://github.com/rust-lang/unsafe-code-guidelines/issues/73

</details>

#### Layout of unions with a single non-zero-sized field

The layout of unions with a single non-[1-ZST]-field" is the same as the
layout of that field if it has no [padding] bytes.

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
because `Zst` is a [1-ZST].

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

the layout of `U1` is **unspecified** because:

* `Zst2` is not a [1-ZST], and
* `SomeOtherStruct` has an unspecified layout and could contain padding bytes.

### C-compatible layout (`#[repr(C)]`)

The layout of `repr(C)` unions follows the C layout scheme.
Per sections [6.5.8.5] and [6.7.2.1.16] of the C11 specification, this means that the offset
of every field is 0, and the alignment of the union is the largest alignment of its fields.
Unsafe code can cast a pointer to the union to a field type to obtain a pointer to any field, and vice versa. 

[6.5.8.5]: http://port70.net/~nsz/c/c11/n1570.html#6.5.8p5
[6.7.2.1.16]: http://port70.net/~nsz/c/c11/n1570.html#6.7.2.1p16

#### Padding

Since all fields are at offset 0, `repr(C)` unions do not have [padding] before
their fields.
They can, however, have padding in each union variant *after* the field, to make
all variants have the same size.

Moreover, the entire union can have tail padding, to make sure the size is a
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

#### Padding Footgun

**Major footgun:** In general, unions can have padding.
On some platform ABIs, such as the popular arm64, C unions may even have [interior padding] *within* fields, where a byte is padding in every variant:

```rust
#[repr(C)]
union U {
   x: (u8, u16), // [u8, 1*pad, u16]
   y: (u8, u8), // [u8, 1*pad, u8, 1*pad]
}
let u = unsafe { mem::zeroed::<U>() }; // resulting bytes: [0, uninit (!!), 0, 0]
let buf: &[u8] = unsafe { slice::from_raw_parts(transmute(&u), 4) }; // UB!
```

This is, surprisingly, undefined behaviour, because it appears that the union is
fully initialized and therefore ought to be castable to a slice.
However, because byte 1 is a padding byte in both variants, it can be a padding
byte in the union type as well. 
Therefore, when the result of `mem::zeroed` is copied onto the stack, the
padding byte is uninitialized, not 0.

This behaviour is platform-specific; on some platforms, this example may be
well-defined.

**C/C++ compatibility hazard:** This footgun exists for compatibility with the
*C/C++ platform ABI, but it is not well-known in C/C++ communities.
In particular, unions are sometimes treated as non-exhaustive, with an expectation that they will be ABI-compatible with future versions of the same code that have additional variatns for the union.
Padding, however, can cause unions not to actually be ABI-compatible with future versions of the same type.
(Note that it's also possible that adding a new variant might change the parameter-passing conventions, however, even in the absence of padding!)
So whenever dealing with a union that might have padding across FFI boundaries, you should be particularly careful not to assume that all bytes are initialized.

<details><summary><b>Rationale</b></summary>

Look. It wasn't our idea.

We could try to limit the blast radius to `extern "C"` functions, but really,
that's just sawing off the end of the footgun.

</details>


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

### Transparent layout (`#[repr(transparent)]`)

`#[repr(transparent)]` is currently unstable for unions, but [RFC 2645]
documents most of its semantics.
Notably, it causes unions to be passed using the same ABI as the non-1-ZST
field.

**Major footgun:** Matching the interior ABI means that all padding bytes of the
*non-1-ZST field will also be padding bytes of the union, so the [interior
*padding footgun] exists with `#[repr(transparent)]` unions.

**Note:** If `U` is a transparent union wrapping a `T`, `U` may not inherit
*`T`'s niches, and therefore `Option<U>` and `Option<T>`, for instance, will not
*necessarily have the same layout or even the same size.

This is because, if `U` contains any zero-sized fields in addition to the `T`
field, the [value model] forces `U` to support uninitialized bytes, and that in
turn prevents `T`'s niches from being present in `U`.
Currently, `U` also supports uninitialized bytes if it does not contain any
additional fields, but it is [**TBD**][#364] if single-field transparent unions
might support niches.

[RFC 2645]: https://github.com/rust-lang/rfcs/blob/master/text/2645-transparent-unions.md

### Bag-o-bytes layout (Repr-raw)

There are applications where it is desirable that unions behave simply as a
buffer of abstract bytes, with no constraints on validity and no interior
padding bytes that can [get surprisingly reset to uninit][interior padding
footgun].

Thus, we propose that Rust support a repr, which we are tentatively calling the Raw-repr, which gives these semantics to unions. The Raw-repr may be `#[repr(Rust)]` or it may be a new repr, say `#[repr(Raw)`], which one is TBD. The Raw-repr will have the following properties:

* All fields are laid out at offset 0.
* The alignment of the union is the greatest alignment among fields (or 1, in the case of an empty union).
* There are no padding bytes---even the bytes that aren't part of any variant, that would otherwise be tail padding, are not padding.
    * If the union is over-aligned with an `#[repr(align(n))]` attribute, then any bytes beyond the "natural" alignment are tail padding.

Note that Raw-repr unions are *not* a substitute for `#[repr(C)]` unions. Although it would be nice if we could avoid the [padding footgun] that way.

<details><summary><b>Rationale</b></summary>

We need at least one repr without the [padding footgun], because interior padding in particular is surprising.
In particular, if users want to treat unions as non-exhaustive in a way that is ABI compatible with future versions with more fields, then such unions must not contain any padding.
The presence of tail padding---such as with `union([u8; 3], u16)`, which could have a single byte of tail padding---is less surprising.
But it would still prevent ABI forwards-compatibility if a `u32` field were added later.

This layout is extremely constrained, so it would generally be against the philosophy of `#[repr(Rust)]` to impose these constraints on the default layout instead of introducing a new one. However, without such constraints, `#[repr(Rust)]` is a just a giant, largely useless footgun, which is a rationale to simply constrain it and leave any potential relaxations, e.g. for safe transmutes and niches, to other reprs. Thus, whether it becomes a new repr or not is still TBD.

</details>

[#354]: https://github.com/rust-lang/unsafe-code-guidelines/issues/354
[#364]: https://github.com/rust-lang/unsafe-code-guidelines/issues/364
[1-ZST]: ../glossary.md#zero-sized-type--zst
[exterior padding]: ../glossary.md#exterior-padding
[interior padding]: ../glossary.md#interior-padding
[layout]: ../glossary.md#layout
[padding]: ../glossary.md#padding
[union values]: ../validity/unions.md#values
[value model]: ../glossary.md#value-model
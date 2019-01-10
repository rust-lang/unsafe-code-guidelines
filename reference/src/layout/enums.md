# Representation of Rust `enum` types

**Disclaimer:** Some parts of this section were decided in RFCs, but
others represent the consensus from issue [#10]. The text will attempt
to clarify which parts are "guaranteed" (owing to the RFC decision)
and which parts are still in a "preliminary" state.

[#10]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/10

## Background

**C-like enums.** The simplest form of enum is simply a list of
variants:

```rust
enum SomeEnum {
  Variant1,
  Variant2,
  Variant3,
```

Such enums are called "C-like" because they correspond quite closely
with enums in the C language (though there are important differences
as well, covered later). Presuming that they have more than one
variant, these sorts of enums are always represented as a simple integer,
though the size will vary.

C-like enums may also specify the value of their discriminants explicitly:

```rust
enum SomeEnum {
  Variant22 = 22,
  Variant44 = 44,
  Variant45,
}
```

As in C, discriminant values that are not specified are defined as
either 0 (for the first variant) or as one more than the prior
variant.

**Data-carrying enums.** Enums whose enums have fields are called
"data-carrying" enums. Note that for the purposes of this definition,
it is not relevant whether those fields are zero-sized. Therefore this
enum is considered "data-carrying":

```rust
enum Foo {
  Bar(()),
  Baz,
}
```

**Option-like enums.** As a special case of data-carrying enums, we
identify "option-like" enums as enums where all of the variants but
one have no fields, and one variant has a single field. The most
common example is `Option` itself. In some cases, as described below,
the compiler may apply special optimization rules to the layout of
option-like enums. The **payload** of an option-like enum is the value
of that single field.

## Enums with a specified representation

Enums may be annotation using the following `#[repr]` tags:

- A specific integer type (called `Int` as a shorthand below):
  - `#[repr(u8)]`
  - `#[repr(u16)]`
  - `#[repr(u32)]`
  - `#[repr(u64)]`
  - `#[repr(i8)]`
  - `#[repr(i16)]`
  - `#[repr(i32)]`
  - `#[repr(i64)]`
- C-compatible layout:
  - `#[repr(C)]`
- C-compatible layout with a specified discriminant size:
  - `#[repr(C, u8)]`
  - `#[repr(C, u16)]`
  - etc

We cover each of the categories below. The layout rules for enums with
explicit `#[repr]` annotations are specified in [RFC 2195][].

[RFC 2195]: https://rust-lang.github.io/rfcs/2195-really-tagged-unions.html

### Layout of an enum with no variants

An enum with no variants can never be instantiated and is logically
equivalent to the "never type" `!`. Such enums are guaranteed to have
the same layout as `!` (zero size and alignment 1).

### Layout of a C-like enum

If there is no `#[repr]` attached to a C-like enum, it is guaranteed
to be represented as an integer of sufficient size to store the
discriminants for all possible variants. The size is selected by the
compiler but must be at least a `u8`.

When a `#[repr(Int)]`-style annotation is attached to a C-like enum
(one without any data for its variants), it will cause the enum to be
represented as a simple integer of the specified size `Int`. This must
be sufficient to store all the required discriminant values.

The `#[repr(C)]` annotation is equivalent, but it selects the same
size as the C compiler would use for the given target for an
equivalent C-enum declaration.

Combining a `C` and `Int` representation (e.g., `#[repr(C, u8)]`) is
not permitted on a C-like enum.

The values used for the discriminant will match up with what is
specified (or automatically assigned) in the enum definition. For
example, the following enum defines the discriminants for its variants
as 22 and 23 respectively:

```rust
enum Foo { 
  // Specificy discriminant of this variant as 22:
  Variant22 = 22,
  
  // Default discriminant is one more than the previous,
  // so 23 will be assigned.
  Variant23
}
```

**Unresolved question:** What about platforms where `-fshort-enums`
are the default? Do we know/care about that?

### Layout for enums that carry data

For enums that carry data, the layout differs depending on whether
C-compatibility is requested or not.

#### Non-C-compatible layouts

When an enum is tagged with `#[repr(Int)]` for some integral type
`Int` (e.g., `#[repr(u8)]`), it will be represented as a C-union of a
series of `#[repr(C)]` structs, one per variant. Each of these structs
begins with an integral field containing the **discriminant**, which
specifies which variant is active. They then contain the remaining
fields associated with that variant.

**Example.** The following enum uses an `repr(u8)` annotation:

```rust
#[repr(u8)]
enum TwoCases {
    A(u8, u16),
    B(u16),
}
```

This will be laid out equivalently to the following more 
complex Rust types:

```
union TwoCasesRepr {
    A: TwoCasesVariantA,
    B: TwoCasesVariantB,
}
        
#[repr(u8)]
enum TwoCasesTag { A, B }

#[repr(C)]
struct TwoCasesVariantA(TwoCasesTag, u8, u16);

#[repr(C)]
struct TwoCasesVariantB(TwoCasesTag, u16);
```

Note that the `TwoCasesVariantA` and `TwoCasesVariantB` structs are
`#[repr(C)]`; this is needed to ensure that the `TwoCasesTag` value
appears at offset 0 in both cases, so that we can read it to determine
the current variant.

#### C-compatible layouts.

When the `#[repr]` tag includes `C`, e.g., `#[repr(C)]` or `#[repr(C,
u8)]`, the layout of enums is changed to better match C++ enums. In
this mode, the data is laid out as a tuple of `(discriminant, union)`,
where `union` represents a C union of all the possible variants. The
type of the discriminant will be the integral type specified (`u8`,
etc) -- if no type is specified, then the compiler will select one
based on what a size a C-like enum would have with the same number of
variants.

This layout, while more compatible and arguably more obvious, is also
less efficient than the non-C compatible layout in some cases in terms
of total size.

**Example.** The following enum:

```rust
#[repr(C, Int)]
enum MyEnum {
    A(u32),
    B(f32, u64),
    C { x: u32, y: u8 },
    D,
}
```

is equivalent to the following Rust definition:

```rust
#[repr(C)]
struct MyEnumRepr {
    tag: MyEnumTag,
    payload: MyEnumPayload,
}

#[repr(Int)]
enum MyEnumTag { A, B, C, D }

#[repr(C)]
union MyEnumPayload {
   A: u32,
   B: MyEnumPayloadB,
   C: MyEnumPayloadC,
   D: (),
}

#[repr(C)]
struct MyEnumPayloadB(f32, u64);

#[repr(C)]
struct MyEnumPayloadC { x: u32, y: u8 }
}
```

This enum can also be represented in C++ as follows:

```c++
#include <stdint.h>

enum class MyEnumTag: CppEquivalentOfInt { A, B, C, D };
struct MyEnumPayloadB { float _0; uint64_t _1;  };
struct MyEnumPayloadC { uint32_t x; uint8_t y; };

union MyEnumPayload {
   uint32_t A;
   MyEnumPayloadB B;
   MyEnumPayloadC C;
};

struct MyEnum {
    MyEnumTag tag;
    MyEnumPayload payload;
};
```

## Enums without a specified representation

If no explicit `#[repr]` attribute is used, then the layout of most
enums is not specified, with one crucial exception: option-like enums
may in some cases use a compact layout that is identical to their
payload.

(Meta-note: The content in this section is not described by any RFC
and is therefore "non-normative".)

### Discriminant elision on Option-like enums

**Definition.** An **option-like enum** is an enum which has:

- one variant with a single field,
- other variants with no fields ("unit" variants).

The simplest example is `Option<T>` itself, where the `Some` variant
has a single field (of type `T`), and the `None` variant has no
fields.  But other enums that fit that same template (and even enums
that include multiple `None`-like fields) fit. 

**Definition.** The **payload** of an option-like enum is the single
field which it contains; in the case of `Option<T>`, the payload has
type `T`.

**Definition.** In some cases, the payload type may contain illegal
values, which are called **niches**. For example, a value of type `&T`
may never be NULL, and hence defines a niche consisting of the
bitstring `0`.  Similarly, the standard library types [`NonZeroU8`]
and friends may never be zero, and hence also define the value of `0`
as a niche. (Types that define niche values will say so as part of the
description of their representation invariant.)

[`NonZeroU8`]: https://doc.rust-lang.org/std/num/struct.NonZeroU8.html

**Option-like enums where the payload defines an adequate number of
niche values are guaranteed to be represented without using any
discriminant at all.** This is called **discriminant elision**.  If
discriminant elision is in effect, then the layout of the enum is
equal to the layout of its payload.

The most common example is that `Option<&u8>` can be represented as an
nullable `&u8` reference -- the `None` variant is then represented
using the niche value zero. This is because a valid `&u8` value can
never be zero, so if we see a zero value, we know that this must be
`None` variant.

In order for the optimization to apply, the payload type must define a
number of niches greater than or equal to the number of unit variants.
In the case of `Option<T>`, this means that any niche at all will
suffice, as there is only one unit variant (`None`).

**Example.** The type `Option<&u32>` will be represented at runtime as
a nullable pointer. FFI interop often depends on this property.

**Example.** As `fn` types are non-nullable, the type `Option<extern
"C" fn()>` will be represented at runtime as a nullable function
pointer (which is therefore equivalent to a C function pointer) . FFI
interop often depends on this property.

**Example.** Consider the following enum definitions:

```rust
enum Enum1<T> {
  Present(T),
  Absent1,
  Absent2,
}

enum Enum2 {
  A, B, C
}  
```

`Enum1<&u8>` is not eligible for discriminant elision, since `&u8`
defines a single niche value, but `Enum1` has two unit
variants. However, `Enum2` has only three legal values (0 for `A`, 1
for `B`, and 2 for `C`), and hence defines a plethora of niche values[^caveat].
Therefore, `Enum1<Enum2>` is guaranteed to be laid out the same as
`Enum2` ([consider the results of applying
`size_of`](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=eadff247f2c5713b8f3b6c9cda297711)).

[^caveat]: Strictly speaking, niche values are considered part of the "representation invariant" for an enum and not its type. Therefore, this section is added only as a preview for future unsafe-code-guidelines discussion.

### Other optimizations

The previous section specified a relatively narrow set of layout
optimizations that are **guaranteed** by the compiler. However, the
compiler is always free to perform **more** optimizations than this
minimal set. For example, the compiler presently treats `Result<T,
()>` and `Option<T>` as equivalent, but this behavior is not
guaranteed to continue as `Result<T, ()>` is not considered
"option-like".

As of this writing, the compiler's current behavior is to attempt to
elide discriminants whenever possible. Furthermore, a variant whose
only fields are of zero-size is considered a unit variant for this
purpose. If eliding discriminants is not possible (e.g., because the
payload does not define sufficient niche values), then the compiler
will select an appropriate discriminant size `N` and use a
representation roughly equivalent to `#[repr(N)]`, though without the
strict `#[repr(C)]` guarantees on each struct. However, this behavior
is not guaranteed to remain the same in future versions of the
compiler and should not be relied upon. (While it is not expected that
existing layout optimizations will be removed, it is possible -- it is
also possible for the compiler to introduce new sorts of
optimizations.)

## Niche values

C-like enums with N variants and no specified representation are
guaranteed to supply niche values corresponding to 256 - N (presuming
that is a positive number). This is because a C-like enum must be
represented using an integer and that integer must correspond to a
valid variant: the precise size of C-like enums is not specified but
it must be at least one byte, which means that there are at least 256
possible bitstrings (only N of which are valid).

Other enums -- or enums with a specified representation -- may supply
niches if their representation invariant permits it, but that is not
**guaranteed**.

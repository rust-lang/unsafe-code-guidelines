# Rust Value Domain

**Note:** This document is not normative nor endorsed by the UCG WG. Its purpose is to be the basis for discussion and to set down some key terminology.

The purpose of this document is to describe what the set of *all possible values* is in Rust.
This is an important definition: one key element of a Rust specification will be to define the [representation relation][representation] of every type.
This relation relates values with lists of bytes: it says, for a given value and list of bytes, if that value is represented by that list.
However, before we can even start specifying the relation, we have to specify the involved domains.
`Byte` is defined as part of [the memory interface][memory-interface]; this document is about defining `Value`.

[representation]: https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/glossary.md#representation
[memory-interface]: memory-interface.md

## Values

The Rust value domain is described by the following (incomplete) type definition (using the `Pointer` type defined by [the memory interface][memory-interface]):

```rust
enum Value {
    /// A mathematical integer, used for `i*`/`u*` types.
    Int(BigInt),
    /// A Boolean value, used for `bool`.
    Bool(bool),
    /// A pointer value, used for (thin) references and raw pointers.
    Ptr(Pointer),
    /// An uninitialized value.
    Uninit,
    /// An n-tuple, used for arrays, structs, tuples, SIMD vectors.
    Tuple(Vec<Self>),
    /// A variant of a sum type, used for enums.
    Variant {
        idx: BigInt,
        data: Box<Self>,
    },
    /// A "bag of raw bytes", used for unions.
    RawBag(Vec<Byte>),
    /* ... */
}
```

The point of this type is to capture the mathematical concepts that are represented by the data we store in memory.
The definition is likely incomplete, and even if it was complete now, we might expand it as Rust grows.
That is okay; all previously defined representation relations are still well-defined when the domain grows, the newly added values will just not be valid for old types as one would expect.

## Example value relations

We show some examples for how one might want to use this `Value` domain to define the value relation for a type.

### `bool`

The value relation for `bool` relates `Bool(b)` with `[r]` if and only if `r.as_int() == Some(if b { 1 } else { 0 })`.
(`as_int` is defined in [the memory interface][memory-interface].)

### `()`

The value relation for the `()` type relates the empty tuple `Tuple([])` (assuming we can use array notation to "match" on `Vec`) with the empty byte list `[]`, and that's it.

### `!`

The value relation for the `!` type is empty: nothing is related to anything at this type.

### `#[repr(C)] struct Pair<T, U>(T, U)`

The value relation for `Pair` is based on the value relations for `T` and `U`.
A value `Tuple([t, u])` is represented by a list of bytes `rt ++ pad1 ++ ru ++ pad2` (using `++` for list concatenation) if:

* `t` is represented by `rt` at type `T`.
* `u` is represented by `ru` at type `U`.
* The length of `rt ++ pad1` is equal to the offset of the `U` field.
* The length of the entire list `rt ++ pad1 ++ ru ++ pad2` is equal to the size of `Pair<T, U>`.

This relation specifies that values of type `Pair` are always 2-tuples (aka, pairs).
It also says that the actual content of the padding bytes is entirely irrelevant, we only care to have the right number of them to "pad" `ru` to the right place and to "pad" the entire list to have the right length.
So, for example when considering `Pair<u8, u16>`, the value `Tuple[42, 119]` is represented on a little-endian target by `[Raw(42), byte, Raw(119), Raw(0)]` for *any* `byte: Byte`.

### `&T`/`&mut T`

Reference types are tricky.
But a possible value relation for sized `T` is:
A value `Ptr(ptr)` is related to `[PtrFragment { ptr, idx: 0 }, ..., PtrFragment { ptr, idx: PTR_SIZE-1 }]` if `ptr` is non-NULL and appropriately aligned (defining alignment is left open for now).

### `u8`

For the value representation of integer types, there are two different reasonable choices.
Certainly, a value `Int(i)` where `i` in `0..256` is related to `[b]` if `b.as_int() == Some(i)`.

And then, maybe, we also want to additionally say that value `Uninit` is related to byte list `[Uninit]`.
This essentially corresponds to saying that uninitialized memory is a valid representation of a `u8` value (namely, the uninitialized value).

### `NonZeroU8`

`NonZeroU8` is basically the same as `u8` except that we remove the 0, and certainly do not consider `Uninit` a valid value.
A value `Int(i)` where `i` in `1..256` is related to `[b]` if `b.as_int() == Some(i)`, and that's it.

### `union`

The `union` type does not even try to interpret memory, so for a `union` of size `n`, the value relation says that for any byte list `bytes` of that length, `RawBag(bytes)` is related to `bytes`.
(Note however that [this definition might not be implementable](https://github.com/rust-lang/unsafe-code-guidelines/issues/156).)

## The role of the value representation in the operational semantics

One key use of the value representation is to define a "typed" interface to memory:

```rust
trait TypedMemory: Memory {
    /// Write a value of the given type to memory.
    fn typed_write(&mut self, ptr: Self::Pointer, val: Value, ty: Type) -> Result;

    /// Read a value of the given type.
    fn typed_read(&mut self, ptr: Self::Pointer, ty: Type) -> Result<Value>;
}
```

Here, `Type` is some representation of the Rust type system (akin to [`Ty`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/type.Ty.html) in the compiler).
This interface is inspired by [Cerberus](https://www.cl.cam.ac.uk/~pes20/cerberus/).

We can implement `TypedMemory` for any `Memory` as follows:
* For `typed_write`, pick any representation of `val` for `ty`, and call `Memory::write`. If no representation exists, we have UB.
* For `typed_read`, read `ty.size()` many bytes from memory, and then determine which value this list of bytes represents. If it does not represent any value, we have UB.

In particular, this defines the semantics of a "typed copy": when an assignment is executed in Rust (at some type `T`), or when an argument of type `T` is passed to a function or a return value of type `T` returned from a function, this is called a "typed copy".
Its semantics is basically to do a `typed_read` at `T` followed by a `typed_write` at `T`.
In particular, this means doing a "typed copy" at `T` of some memory that has no valid representation at `T` is undefined behavior!
This also means that for types that have padding, the "typed copy" does not preserve the padding bytes.

## Relation to validity invariant

One way we *could* also use the value representation (and the author thinks this is exceedingly elegant) is to define the validity invariant.
Certainly, it is the case that if a list of bytes is not related to any value for a given type `T`, then that list of bytes is *invalid* for `T` and it should be UB to produce such a list of bytes at type `T`.
We could decide that this is an "if and only if", i.e., that the validity invariant for a type is exactly "must be in the value representation".
For many types this is likely what we will do anyway (e.g., for `bool` and `!` and `()` and integers), but for references, this choice would mean that *validity of the reference cannot depend on what memory looks like*---so "dereferencable" and "points to valid data" cannot be part of the validity invariant for references.
The reason this is so elegant is that, as we have seen above, a "typed copy" already very naturally is UB when the memory that is copied is not a valid representation of `T`.
This means we do not even need a special clause in our specification for the validity invariant---in fact, the term does not even have to appear in the specification---as everything juts falls out of how a "typed copy" applies the value representation twice.

Justifying the `dereferencable` LLVM attribute is, in this case, left to the aliasing model (e.g. [Stacked Borrows]), just like the `noalias` attribute.

[Stacked Borrows]: stacked-borrows.md

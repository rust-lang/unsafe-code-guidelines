# Representation of Boolean, Floating Point, and Integral Types
This chapter represents the consensus from issue [#9]. It documents the memory layout and considerations for `bool`, `usize`, `isize`, floating point types, and integral types.
[#9]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/9

## Overview
These are all scalar types, representing a single value. These types have no representation variants (no #[repr(C)] or #[repr(Rust)]. Their size is fixed and well-defined across FFI boundaries and map to their corresponding integral types in the C ABI.
- `bool`: 1 byte
  - any `bool` can be cast into an integer, taking on the values 1 (true) or 0 (false)
- `usize`, `isize`: pointer-sized unsigned/signed integer type
- `u8` .. `u128`, `i8` .. `i128`
  - {8, 16, 32, 64, 128}-bit unsigned integer
  - {8, 16, 32, 64, 128}-bit signed integer
- `f32`, `f64`
  - IEEE floats
  - 32-bit or 64-bit
- `char`
  - C++ char: equivalent to either `i8`/`u8`
  - Rust char: 32-bit
  - not ABI compatible
  - represents [Unicode scalar value](http://www.unicode.org/glossary/#unicode_scalar_value)

##`usize`/`isize`
Types `usize` and `isize` are committed to having the same size as a native pointer on the platform. The representation of `usize` determines the following:
- how much a pointer of a certain type can be offseted,
- the maximum size of Rust objects (because size_of/size_of_val return `usize`),
- the maximum number of elements in an array ([T; N: usize]),
- `usize`/`isize` in C FFI are compatible with C's `uintptr_t` / `intptr_t` (and have the same size and alignment).

The maximum size of any single value must fit within `usize` to [ensure that pointer diff is representable](https://github.com/rust-rfcs/unsafe-code-guidelines/pull/5#discussion_r212703192).

`usize` and C’s `unsized` are *not* equivalent.

## Booleans
Rust's `bool` has the same layout as C17's` _Bool`, that is, its size and alignment are implementation-defined.

Note: on all platforms that Rust's currently supports, the size and alignment of bool are 1, and its ABI class is INTEGER.

For full ABI compatibility details, see [Gankro’s post](https://gankro.github.io/blah/rust-layouts-and-abis/#the-layoutsabis-of-builtins).

## Relationship to C integer hierarchy
C integers:
- char: at least 8 bits
- short: at least 16 bits (also at least a char)
- int: at least a short (intended to be a native integer size)
- long: at least 32 bits (also at least an int)
- long long: at least 64 bits (also at least a long)
The C integer types specify a minimum size, but not the exact size. For this reason, Rust integer types are not necessarily compatible with the “corresponding” C integer type. Instead, use the corresponding fixed size data types (e.g. `i64` in Rust would correspond to `int64_t` in C).

## Controversies
There has been some debate about what to pick as the "official" behavior for bool:
* Rust does what C does (this is what the lang team decided)
    * and in all cases you care about, that is 1 byte that is 0 or 1
or
* Rust makes it 1 byte with values 0 or 1
    * and in all cases you care about, this is what C does
    
Related discussions: [document the size of bool](https://github.com/rust-lang/rust/pull/46156), [bool== _Bool?](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/53#issuecomment-447050232), [bool ABI](https://github.com/rust-lang/rust/pull/46176#issuecomment-359593446)
    



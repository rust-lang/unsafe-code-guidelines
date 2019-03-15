# Layout of Boolean, Floating Point, and Integral Types

This chapter represents the consensus from issue [#9]. It documents the memory
layout and considerations for `bool`, floating point types (`f{32, 64}`), and
integral types (`{i,u}{8,16,32,64,128,size}`).

These types are all scalar types, representing a single value, and have no
layout `#[repr()]` flags.

[#9]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/9

## `bool`

Rust's `bool` has the same layout as C17's` _Bool`, that is, its size and
alignment are implementation-defined. Any `bool` can be cast into an integer,
taking on the values 1 (`true`) or 0 (`false`).

> **Note**: on all platforms that Rust's currently supports, its size and
> alignment are 1, and its ABI class is `INTEGER` - see [Rust Layout and ABIs].

[Rust Layout and ABIs]: https://gankro.github.io/blah/rust-layouts-and-abis/#the-layoutsabis-of-builtins

## `char`

Rust char is 32-bit wide and represents [Unicode scalar value](http://www.unicode.org/glossary/#unicode_scalar_value).

> **Note**: Rust `char` type is not layout compatible with C / C++ `char` types.
> The C / C++ `char` types correspond to either Rust's `i8` or `u8` types on all
> currently supported platforms, depending on their signedness. Rust does not
> support C platforms in which C `char` is not 8-bit wide.

## `usize`/`isize`

The `usize` and `isize` types are pointer-sized signed and unsigned integers.
They have the same layout as the [pointer types] for which the pointee is
`Sized`, and are layout compatible with C's `uintptr_t` and `intptr_t` types.

> **Note**: Rust's `usize` and C's `unsigned` types are **not** equivalent. C's
> `unsigned` is at least as large as a short, allowed to have padding bits, etc.
> but it is not necessarily pointer-sized.

The layout of `usize` determines the following:

- the maximum size of Rust objects (`size_of` and `size_of_val` return `usize`),
- the maximum number of elements in an array (`[T; N: usize]`),
- how much a pointer of a certain type can be offseted (limited by `usize::max_value()`).

> **FIXME**: Pointer `add` operates on `usize`, but pointer `offset` operates on
> `isize`, so unless by "offseted" we mean something different from `ptr.offset`
> above, `usize::max_value()` does not determine how much a pointer can be
> "offseted". We should probably be more specific here and call out `ptr.add`
> and `ptr.offset` explicitly.

The maximum size of any single value must fit within `usize` to [ensure that
pointer diff is
representable](https://github.com/rust-rfcs/unsafe-code-guidelines/pull/5#discussion_r212703192).

> **FIXME**: This does not make sense. We state that the layout of `usize`
> determines the maximum size of an object, and then argue that this is to
> ensure that pointer diff is representable, which won't be the case if the size
> of an object is `usize::max_val()`. The link cited actually states that, right
> now, the largest size of a Rust object is limited by `isize::max_value()`.

[pointer types]: ./pointers.md

## Fixed-width integer types

Rust's signed and unsigned fixed-width integer types `{i,u}{8,16,32,64}` have
the same layout as the C fixed-width integer types from the `<stdint.h>` header
`{u,}int{8,16,32,64}_t`. That is:

* these types have no padding bits,
* their size exactly matches their bit-width,
* negative values of signed integer types are represented using 2's complement.

This properties also hold for Rust's 128-bit wide `{i,u}128` integer types, but
C does not expose equivalent types in `<stdint.h>`.

Rust fixed-width integer types are therefore safe to use directly in C FFI where 
the corresponding C fixed-width integer types are expected.
integer types are expected.

### Layout compatibility with C native integer types

The specification of native C integer types, `char`, `short`, `int`, `long`,
... as well as their `unsigned` variants, has a lower bound on their  size,
e.g., `short` is _at least_ 16-bit wide and _at least_ as wide as `char`.
Their actual exact sizes are _implementation-defined_. 

Libraries like `libc` use knowledge of this _implementation-defined_ behavior on
each platform to select a layout-compatible Rust fixed-width integer type when
interfacing with native C integer types.

> **Note**: Rust does not support C platforms on which the C native integer type
> are not compatible with any of Rust's fixed-width integer type (e.g. because
> of padding-bits, lack of 2's complement, etc.).

## Fixed-width floating point types

Rust's `f32` and `f64` single (32-bit) and double (64-bit) precision
floating-point types have [IEEE-754] `binary32` and `binary64` floating-point
layouts, respectively.

When the platforms' `"math.h"` header defines the `__STDC_IEC_559__` macro,
Rust's floating-point types are safe to use directly in C FFI where the
appropriate C types are expected (`f32` for `float`, `f64` for `double`).

If the C platform's `"math.h"` header does not define the `__STDC_IEC_559__`
macro, whether using `f32` and `f64` in C FFI is safe or not for which C type is
_implementation-defined_.

> **Note**: the `libc` crate uses knowledge of each platform's
> _implementation-defined_ behavior to provide portable `libc::c_float` and
> `libc::c_double` types that can be used to safely interface with C via FFI.

[IEEE-754]: https://en.wikipedia.org/wiki/IEEE_754

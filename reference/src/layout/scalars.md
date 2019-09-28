# Layout of scalar types

This chapter represents the consensus from issue [#9]. It documents the memory
layout and considerations for `bool`, `char`, floating point types (`f{32, 64}`), and integral types (`{i,u}{8,16,32,64,128,size}`).

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

Rust char is 32-bit wide and represents an [unicode scalar value]. The alignment
of `char` is _implementation-defined_.

[unicode scalar value]: http://www.unicode.org/glossary/#unicode_scalar_value

> **Note**: Rust `char` type is not layout compatible with C / C++ `char` types.
> The C / C++ `char` types correspond to either Rust's `i8` or `u8` types on all
> currently supported platforms, depending on their signedness. Rust does not
> support C platforms in which C `char` is not 8-bit wide.

## `isize` and `usize`

The `isize` and `usize` types are pointer-sized signed and unsigned integers.
They have the same layout as the [pointer types] for which the pointee is
`Sized`, and are layout compatible with C's `uintptr_t` and `intptr_t` types.

> **Note**: As a result, Rust's `usize` is **not** necessarily layout
> compatible with C's `size_t`, since `uintptr_t` may be different from
> `size_t`.

> **Note**: C99 [7.18.2.4](https://port70.net/~nsz/c/c99/n1256.html#7.18.2.4)
> requires `uintptr_t` and `intptr_t` to be at least 16-bit wide. All 
> platforms we currently support have a C platform, and as a consequence,
> `isize`/`usize` are at least 16-bit wide for all of them.

> **Note**: in the current Rust implementation, the layouts of `isize` and
> `usize` determine the following:
> 
> * the maximum size of Rust _allocations_ is limited to `isize::max_value()`.
>   The LLVM `getelementptr` instruction uses signed-integer field offsets. Rust
>   calls `getelementptr` with the `inbounds` flag which assumes that field
>   offsets do not overflow,
>
> * the maximum number of elements in an array is `usize::max_value()` (`[T; N:
>   usize]`. Only ZST arrays can probably be this large in practice, non-ZST
>   arrays are bound by the maximum size of Rust values,
>
> * the maximum value in bytes by which a pointer can be offseted using
>   `ptr.add` or `ptr.offset` is `isize::max_value()`.
>
> These limits have not gone through the RFC process and are not guaranteed to
> hold.

[pointer types]: ./pointers.md

## Fixed-width integer types

For all Rust's fixed-width integer types `{i,u}{8,16,32,64,128}` it holds that:

* these types have no padding bits,
* their size exactly matches their bit-width,
* negative values of signed integer types are represented using 2's complement.

Furthermore, Rust's signed and unsigned fixed-width integer types
`{i,u}{8,16,32,64}` have the same layout the C fixed-width integer types from
the `<stdint.h>` header `{u,}int{8,16,32,64}_t`. These fixed-width integer types
are therefore safe to use directly in C FFI where the corresponding C
fixed-width integer types are expected.

The alignment of Rust's `{i,u}128` is _unspecified_ and allowed to change.

> **Note**: While the C standard does not define fixed-width 128-bit wide
> integer types, many C compilers provide non-standard `__int128` types as a
> language extension. The layout of `{i,u}128` in the current Rust
> implementation does **not** match that of these C types, see 
> [rust-lang/#54341](https://github.com/rust-lang/rust/issues/54341).

### Layout compatibility with C native integer types

The specification of native C integer types, `char`, `short`, `int`, `long`,
... as well as their `unsigned` variants, guarantees a lower bound on their  size,
e.g., `short` is _at least_ 16-bit wide and _at least_ as wide as `char`.

Their exact sizes are _implementation-defined_. 

Libraries like `libc` use knowledge of this _implementation-defined_ behavior on
each platform to select a layout-compatible Rust fixed-width integer type when
interfacing with native C integer types (e.g. `libc::c_int`).

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

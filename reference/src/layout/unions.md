# Layout of unions

**Disclaimer:** This chapter represents the consensus from issue
[#13].  The statements in here are not (yet) "guaranteed"
not to change until an RFC ratifies them.

[#13]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/13

The only degree of freedom the compiler has when computing the layout of a union
like

```rust,ignore
union U { f1: T1, f2: T2 }
```

is to determine the offset of the fields.  The layout of these fields themselves
is already entirely determined by their types, and since we intend to allow
creating references to fields (`&u.f1`), unions do not have any wiggle-room
there.

### Default layout ("repr rust")

**The default layout of unions is not specified.** As of this writing, we want
to keep the option of using non-zero offsets open for the future; whether this
is useful depends on what exactly the compiler-assumed invariants about union
contents are.

Even if the offsets happen to be all 0, there might still be differences in the
function call ABI.  If you need to pass unions by-value across an FFI boundary,
you have to use `#[repr(C)]`.

### C-compatible layout ("repr C")

The layout of `repr(C)` unions follows the C layout scheme. Per sections
[6.5.8.5] and [6.7.2.1.16] of the C11 specification, this means that the offset
of every field is 0. Unsafe code can cast a pointer to the union to a field type
to obtain a pointer to any field, and vice versa. 

[6.5.8.5]: http://port70.net/~nsz/c/c11/n1570.html#6.5.8p5
[6.7.2.1.16]: http://port70.net/~nsz/c/c11/n1570.html#6.7.2.1p16

#### Padding

Since all fields are at offset 0, `repr(C)` unions do not have padding before
their fields. They can, however, have trailing padding, to make sure the size is
a multiple of the alignment:

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

If a `#[repr(C)]` union contains a field of zero-size, that field does not
occupy space in Rust unions. For example:

```rust
# use std::mem::{size_of, align_of};
#[repr(C)] 
union U {
  x: u8,
  y: (),
}
# fn main() {
assert_eq!(size_of::<U>(), 1);
# }
```

The field does, however, participate in the layout computation of the union, and
can raise its alignment requirement, which in turn can introduce trailing
padding. For example:

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
// This introduces trailing padding, raising the union size to 2
assert_eq!(size_of::<U>(), 2);
# }
```

This handling of zero-sized types is equivalent to the handling of zero-sized
types in struct fields, and matches the behavior of GCC and Clang for unions in
C when zero-sized types are allowed via their language extensions.

**C++ compatibility hazard**: C++ does, in general, give a size of 1 to empty
structs. If an empty struct in C++ is used as an union field, a "naive"
translation of that code into Rust will not produce a compatible result.

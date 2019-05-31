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

### C-compatible layout ("repr C")

For unions tagged `#[repr(C)]`, the compiler will apply the C layout scheme.  Per
sections [6.5.8.5] and [6.7.2.1.16] of the C11 specification, this means that
the offset of every field is 0.  Unsafe code can cast a pointer to the union to
a field type to obtain a pointer to any field, and vice versa.

[6.5.8.5]: http://port70.net/~nsz/c/c11/n1570.html#6.5.8p5
[6.7.2.1.16]: http://port70.net/~nsz/c/c11/n1570.html#6.7.2.1p16

### Default layout ("repr rust")

**The default layout of unions is not specified.** As of this writing, we want
to keep the option of using non-zero offsets open for the future; whether this
is useful depends on what exactly the compiler-assumed invariants about union
contents are.

Even if the offsets happen to be all 0, there might still be differences in the
function call ABI.  If you need to pass unions by-value across an FFI boundary,
you have to use `#[repr(C)]`.

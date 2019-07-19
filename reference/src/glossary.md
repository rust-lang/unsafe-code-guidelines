## Glossary

#### Aliasing

(Please note: a full aliasing model for Rust has not yet been constructed, but
at the moment we can give the following definition.)

*Aliasing* is any time one pointer or reference points to a "span" of memory
that overlaps with the span of another pointer or reference. A span of memory is
similar to how a slice works: there's a base byte address as well as a length in
bytes.

Consider the following example:

```rust
fn main() {
    let u: u64 = 7_u64;
    let r: &u64 = &u;
    let s: &[u8] = unsafe {
        core::slice::from_raw_parts(&u as *const u64 as *const u8, 8)
    };
    let (head, tail) = s.split_first().unwrap();
}
```

In this case, both `r` and `s` alias each other, since they both point to all of
the bytes of `u`.

However, `head` and `tail` do not alias each other: `head` points to the first
byte of `u` and `tail` points to the other seven bytes of `u` after it.

* The span length of `&T`, `&mut T`, `*const T`, or `*mut T` when `T` is
  [`Sized`](https://doc.rust-lang.org/core/marker/trait.Sized.html) is
  `size_of<T>()`.
* When `T` is not `Sized` the span length is `size_of_val(t)`.

One interesting side effect of these rules is that references and pointers to
Zero Sized Types _never_ alias each other, because their span length is always 0
bytes.

It is also important to know that LLVM IR has a `noalias` attribute that works
somewhat differently from this definition. However, that's considered a low
level detail of a particular Rust implementation. When programming Rust, the
Abstract Rust Machine is intended to operate according to the definition here.

#### Interior mutability

*Interior Mutation* means mutating memory where there also exists a live shared reference pointing to the same memory; or mutating memory through a pointer derived from a shared reference.
"live" here means a value that will be "used again" later.
"derived from" means that the pointer was obtained by casting a shared reference and potentially adding an offset.
This is not yet precisely defined, which will be fixed as part of developing a precise aliasing model.

Finding live shared references propagates recursively through references, but not through raw pointers.
So, for example, if data immediately pointed to by a `&T` or `& &mut T` is mutated, that's interior mutability.
If data immediately pointed to by a `*const T` or `&*const T` is mutated, that's *not* interior mutability.

*Interior mutability* refers to the ability to perform interior mutation without causing UB.
All interior mutation in Rust has to happen inside an [`UnsafeCell`](https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html), so all data structures that have interior mutability must (directly or indirectly) use `UnsafeCell` for this purpose.

#### Validity and safety invariant

The *validity invariant* is an invariant that all data must uphold any time it is accessed or copied in a typed manner.
This invariant is known to the compiler and exploited by optimizations such as improved enum layout or eliding in-bounds checks.

In terms of MIR statements, "accessed or copied" means whenever an assignment statement is executed.
That statement has a type (LHS and RHS must have the same type), and the data being assigned must be valid at that type.
Moreover, arguments passed to a function must be valid at the type given in the callee signature, and the return value of a function must be valid at the type given in the caller signature.
OPEN QUESTION: Are there more cases where data must be valid?

In terms of code, some data computed by `TERM` is valid at type `T` if and only if the following program does not have UB:
```rust,ignore
fn main() { unsafe {
  let t: T = std::mem::transmute(TERM);
} }
```

The *safety* invariant is an invariant that safe code may assume all data to uphold.
This invariant is used to justify which operations safe code can perform.
The safety invariant can be temporarily violated by unsafe code, but must always be upheld when interfacing with unknown safe code.
It is not relevant when arguing whether some *program* has UB, but it is relevant when arguing whether some code safely encapsulates its unsafety -- IOW, it is relevant when arguing whether some *library* can be used by safe code to *cause* UB.

In terms of code, some data computed by `TERM` (possibly constructed from some `arguments` that can be *assumed* to satisfy the safety invariant) is valid at type `T` if and only if the following library function can be safely exposed to arbitrary (safe) code as part of the public library interface:
```rust,ignore
pub fn make_something(arguments: U) -> T { unsafe {
  std::mem::transmute(TERM)
} }
```

One example of valid-but-unsafe data is a `&str` or `String` that's not well-formed UTF-8: the compiler will not run its own optimizations that would cause any trouble here, so unsafe code may temporarily violate the invariant that strings are `UTF-8`.
However, functions on `&str`/`String` may assume the string to be `UTF-8`, meaning they may cause UB if the string is *not* `UTF-8`.
This means that unsafe code violating the UTF-8 invariant must not perform string operations (it may operate on the data as a byte slice though), or else it risks UB.
Moreover, such unsafe code must not return a non-UTF-8 string to the "outside" of its safe abstraction boundary, because that would mean safe code could cause UB by doing `bad_function().chars().count()`.

To summarize: *Data must always be valid, but it only must be safe in safe code.*
For some more information, see [this blog post](https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html).

#### Layout

The *layout* of a type defines its size and alignment as well as the offsets of its subobjects (e.g. fields of structs/unions/enum/... or elements of arrays).
Moreover, the layout of a type records its *function call ABI* (or just *ABI* for short): how the type is passed *by value* across a function boundary.

Note: Originally, *layout* and *representation* were treated as synonyms, and Rust language features like the `#[repr]` attribute reflect this. 
In this document, *layout* and *representation* are not synonyms.

#### Niche

The *niche* of a type determines invalid bit-patterns that will be used by layout optimizations.

For example, `&mut T` has at least one niche, the "all zeros" bit-pattern. This
niche is used by layout optimizations like ["`enum` discriminant
elision"](layout/enums.html#discriminant-elision-on-option-like-enums) to
guarantee that `Option<&mut T>` has the same size as `&mut T`.

While all niches are invalid bit-patterns, not all invalid bit-patterns are
niches. For example, the "all bits uninitialized" is an invalid bit-pattern for
`&mut T`, but this bit-pattern cannot be used by layout optimizations, and is not a
niche.


### TODO

* *tag*
* *rvalue*
* *lvalue*
* *representation*
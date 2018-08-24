# Data structure representation and validity requirements

## Introduction

This discussion is meant to focus on two things:

- What guarantees does Rust make regarding the layout of data structures?
- What invariants does the compiler require from the various Rust types?
  - the "validity invariant", as defined in [Ralf's blog post][bp]

NB. The discussion is **not** meant to discuss the "safety invariant"
from [Ralf's blog post][bp], as that can be handled later.

[bp]: https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html

### Layout of data structures

In general, Rust makes few guarantees about the memory layout of your
structures. For example, by default, the compiler has the freedom to
rearrange the field order of your structures for more efficiency (as
of this writing, we try to minimize the overall size of your
structure, but this is the sort of detail that can easily change). For
safe code, of course, any rearrangements "just work" transparently.

If, however, you need to write unsafe code, you may wish to have a
fixed data structure layout. In that case, there are ways to specify
and control how an individual struct will be laid out -- notably with
`#[repr]` annotations. One purpose of this section, then, is to layout
what sorts of guarantees we offer when it comes to layout, and also
what effect the various `#[repr]` annotations have.

### Validity invariant

The "validity invariant" for each type defines what must hold whenever
a value of this type is considered to be initialized. The compiler expects
the validity invariant to hold **at all times** and is thus allowed to use
these invariants to (e.g.) affect the layout of data structures or do other
optimizations.

Therefore, the validity invariant must **at minimum** justify all the
layout optimizations that the compiler does. We may want a stronger
invariant, however, so as to leave room for future optimization.

As an example, a value of `&T` type can never be null -- therefore,
`Option<&T>` can use null to represent `None`.

## Goals

- Define what we guarantee about the layout of various types
  and the effect of `#[repr]` annotations.
- Define the **validity requirements** of various types that unsafe
  programmers must uphold at all times.
    - Also examine when/how we could dynamically check these requirements.
- Uncover the sorts of constraints that we may wish to satisfy in the
  future.

## Some interesting examples and questions

- `&T` where `T: Sized`
  - This is **guaranteed** to be a non-null pointer
- `Option<&T>` where `T: Sized`
  - This is **guaranteed** to be a nullable pointer
- `Option<extern "C" fn()>`
- `usize`
  - Platform dependent size, but guaranteed to be able to store a pointer?
  - Also an array length?
- Uninitialized bits -- for which types are uninitialized bits valid?
- If you have `struct A { .. }` and `struct B { .. }` with no
  `#[repr]` annotations, and they have the same field types, can we
  say that they will have the same layout?
  - or do we have the freedom to rearrange the types of `A` but not
    `B`, e.g. based on PGO results

## Active threads

To start, we will create threads for each major categories of types
(with a few suggested focus points):

- Integers and floating points
    - What about uninitialized values?
- Booleans
    - Prior discussions ([#46156][], [#46176][]) documented bool as a single
      byte that is either 0 or 1.
- Enums
    - See dedicated thread about "niches" and `Option`-style layout optimization
      below.
    - Define: C-like enum
    - Can a C-like enum ever have an invalid discriminant? (Presumably not)
    - Empty enums and the `!` type
    - [RFC 2195][] defined the layout of `#[repr(C)]` enums with payloads.
    - [RFC 2363][] offers a proposal to permit specifying discriminations.
- Structs
    - Do we ever say *anything* about how a `#[repr(rust)]` struct is laid out?
      - e.g., what about different structs with same definition
      - across executions of the same program?
- Tuples
    - Are these effectively anonymous structs? 
- Unions
    - Can we ever say anything about the initialized contents of a union?
    - Is `#[repr(C)]` meaningful on a union?
- Fn pointers (`fn()`, `extern "C" fn()`)
- References `&T` and `&mut T`
    - Out of scope: aliasing rules
    - We currently tell LLVM they are aligned and dereferenceable, have to justify that
    - Safe code may use them also
- Raw pointers
    - Effectively same as integers?
- Representation knobs:
    - Custom alignment ([RFC 1358])
    - Packed ([RFC 1240] talks about some safety issues)
- ... what else?

We will also create categories for the following specific areas:

- Niches: Optimizing `Option`-like enums
- Uninitialized memory: when/where are uninitializes values permitted, if ever?
- ... what else?


[#46156]: https://github.com/rust-lang/rust/pull/46156
[#46176]: https://github.com/rust-lang/rust/pull/46176
[RFC 2363]: https://github.com/rust-lang/rfcs/pull/2363
[RFC 2195]: https://rust-lang.github.io/rfcs/2195-really-tagged-unions.html
[RFC 1358]: https://rust-lang.github.io/rfcs/1358-repr-align.html
[RFC 1240]: https://rust-lang.github.io/rfcs/1240-repr-packed-unsafe-ref.html

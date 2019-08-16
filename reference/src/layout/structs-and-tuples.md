# Layout of structs and tuples

**Disclaimer:** This chapter represents the consensus from issues
[#11] and [#12].  The statements in here are not (yet) "guaranteed"
not to change until an RFC ratifies them.

[#11]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11
[#12]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/12

## Tuple types

In general, an anonymous tuple type `(T1..Tn)` of arity N is laid out
"as if" there were a corresponding tuple struct declared in libcore:

```rust,ignore
#[repr(Rust)]
struct TupleN<P1..Pn:?Sized>(P1..Pn);
```

In this case, `(T1..Tn)` would be compatible with `TupleN<T1..Tn>`.
As discussed below, this generally means that the compiler is **free
to re-order field layout** as it wishes. Thus, if you would like a
guaranteed layout from a tuple, you are generally advised to create a
named struct with a `#[repr(C)]` annotation (see [the section on
structs for more details](#structs)).

Note that the final element of a tuple (`Pn`) is marked as `?Sized` to
permit unsized tuple coercion -- this is implemented on nightly but is
currently unstable ([tracking issue][#42877]). In the future, we may
extend unsizing to other elements of tuples as well.

[#42877]: https://github.com/rust-lang/rust/issues/42877

### Other notes on tuples

Some related discussion:

- [RFC #1582](https://github.com/rust-lang/rfcs/pull/1582) proposed
  that tuple structs should have a "nested layout", where
  e.g. `(T1, T2, T3)` would in fact be laid out as `(T1, (T2,
  T3))`. The purpose of this was to permit variadic matching and so
  forth against some suffix of the struct. This RFC was not accepted,
  however. This layout requires extra padding and seems somewhat
  surprising: it means that the layout of tuples and tuple structs
  would diverge significantly from structs with named fields.

<a name="structs"></a>

## Struct types

Structs come in two principle varieties:

```rust,ignore
// Structs with named fields
struct Foo { f1: T1, .., fn: Tn }

// Tuple structs
struct Foo(T1, .., Tn);
```

In terms of their layout, tuple structs can be understood as
equivalent to a named struct with fields named `0..n-1`:

```rust,ignore
struct Foo {
  0: T1,
  ...
  n-1: Tn
}
```

(In fact, one may use such field names in patterns or in accessor
expressions like `foo.0`.)

Structs can have various `#[repr]` flags that influence their layout:

- `#[repr(Rust)]` -- the default.
- `#[repr(C)]` -- request C compatibility
- `#[repr(align(N))]` -- specify the alignment
- `#[repr(packed)]` -- request packed layout where fields are not internally aligned
- `#[repr(transparent)]` -- request that a "wrapper struct" be treated
  "as if" it were an instance of its field type when passed as an
  argument

### Default layout ("repr rust")

**The default layout of structs is not specified.** As of this
writing, we have not reached a full consensus on what limitations
should exist on possible field struct layouts, so effectively one must
assume that the compiler can select any layout it likes for each
struct on each compilation, and it is not required to select the same
layout across two compilations. This implies that (among other things)
two structs with the same field types may not be laid out in the same
way (for example, the hypothetical struct representing tuples may be
laid out differently from user-declared structs).

Known things that can influence layout (non-exhaustive):

- the type of the struct fields and the layout of those types
- compiler settings, including esoteric choices like optimization fuel

**A note on determinism.** The definition above does not guarantee
determinism between executions of the compiler -- two executions may
select different layouts, even if all inputs are identical. Naturally,
in practice, the compiler aims to produce deterministic output for a
given set of inputs. However, it is difficult to produce a
comprehensive summary of the various factors that may affect the
layout of structs, and so for the time being we have opted for a
conservative definition.

**Compiler's current behavior.** As of the time of this writing, the
compiler will reorder struct fields to minimize the overall size of
the struct (and in particular to eliminate padding due to alignment
restrictions).

Layout is presently defined not in terms of a "fully monomorphized"
struct definition but rather in terms of its generic definition along
with a set of substitutions (values for each type parameter; lifetime
parameters do not affect layout). This distinction is important
because of *unsizing* -- if the final field has generic type, the
compiler will not reorder it, to allow for the possibility of
unsizing. E.g., `struct Foo { x: u16, y: u32 }` and `struct Foo<T> {
x: u16, y: T }` where `T = u32` are not guaranteed to be identical.

#### Zero-sized structs

For `repr(Rust)`, `repr(packed(N))`, `repr(align(N))`, and `repr(C)`
structs: if all fields of a struct have size 0, then the struct has size 0.

For example, all these types are zero-sized:

```rust
# use std::mem::size_of;
#[repr(align(32))] struct Zst0;
#[repr(C)] struct Zst1(Zst0);
struct Zst2(Zst1, Zst0);
# fn main() {
# assert_eq!(size_of::<Zst0>(), 0);
# assert_eq!(size_of::<Zst1>(), 0);
# assert_eq!(size_of::<Zst2>(), 0);
# }
```

#### Default layout of structs where only a single field is not a 1-ZST

The default layout of structs where only a single field is not a 1-ZST is the
same as the layout of that non-1-ZST field.

For example, the layout of:

```rust
struct SomeStruct(i32, ());
```

is the same as the layout of `i32`, but the layout of:

```rust
#[repr(align(16))] struct Zst;
struct SomeOtherStruct(i32, Zst);
```
is **unspecified**, since `Zst` is not a [1-ZST].

#### Unresolved questions

During the course of the discussion in [#11] and [#12], various
suggestions arose to limit the compiler's flexibility. These questions
are currently considering **unresolved** and -- for each of them -- an
issue has been opened for further discussion on the repository. This
section documents the questions and gives a few light details, but the
reader is referred to the issues for further discussion.

**Zero-sized structs ([#37]).** If you have a struct which --
transitively -- contains no data of non-zero size, then the size of
that struct will be zero as well. These zero-sized structs appear
frequently as exceptions in other layout considerations (e.g.,
single-field structs). An example of such a struct is
`std::marker::PhantomData`.

[#37]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/37

**Homogeneous structs ([#36]).** If you have homogeneous structs, where all
the `N` fields are of a single type `T`, can we guarantee a mapping to
the memory layout of `[T; N]`? How do we map between the field names
and the indices? What about zero-sized types?

[#36]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/36

**Deterministic layout ([#35]).** Can we say that layout is some deterministic
function of a certain, fixed set of inputs? This would allow you to be
sure that if you do not alter those inputs, your struct layout would
not change, even if it meant that you can't predict precisely what it
will be. For example, we might say that struct layout is a function of
the struct's generic types and its substitutions, full stop -- this
would imply that any two structs with the same definition are laid out
the same. This might interfere with our ability to do profile-guided
layout or to analyze how a struct is used and optimize based on
that. Some would call that a feature.

[#35]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/35

### C-compatible layout ("repr C")

For structs tagged `#[repr(C)]`, the compiler will apply a C-like
layout scheme. See section 6.7.2.1 of the [C17 specification][C17] for
a detailed write-up of what such rules entail (as well as the relevant
specs for your platform). For most platforms, however, this means the
following:

[C17]: http://www.open-std.org/jtc1/sc22/wg14/www/abq/c17_updated_proposed_fdis.pdf

- Field order is preserved.
- The first field begins at offset 0.
- Assuming the struct is not packed, each field's offset is aligned[^aligned] to
  the ABI-mandated alignment for that field's type, possibly creating
  unused padding bits.
- The total size of the struct is rounded up to its overall alignment.  

[^aligned]: Aligning an offset O to an alignment A means to round up the offset O until it is a multiple of the alignment A.

The intention is that if one has a set of C struct declarations and a
corresponding set of Rust struct declarations, all of which are tagged
with `#[repr(C)]`, then the layout of those structs will all be
identical. Note that this setup implies that none of the structs in
question can contain any `#[repr(Rust)]` structs (or Rust tuples), as
those would have no corresponding C struct declaration -- as
`#[repr(Rust)]` types have undefined layout, you cannot safely declare
their layout in a C program.

See also the notes on [ABI compatibility](#fnabi) under the section on `#[repr(transparent)]`.

**Structs with no fields.** One area where Rust layout can deviate
from C/C++ -- even with `#[repr(C)]` -- comes about with "empty
structs" that have no fields. In C, an empty struct declaration like
`struct Foo { }` is illegal. However, both gcc and clang support
options to enable such structs, and [assign them size
zero](https://godbolt.org/z/AS2gdC). Rust behaves the same way --
empty structs have size 0 and alignment 1 (unless an explicit
`#[repr(align)]` is present). C++, in contrast, gives empty structs a
size of 1, unless they are inherited from or they are fields that have
the `[[no_unique_address]]` attribute, in which case they do not
increase the overall size of the struct.

**Structs of zero-size.** It is also possible to have structs that
have fields but still have zero size. In this case, the size of the
struct would be zero, but its alignment may be greater. For example,
`#[repr(C)] struct Foo { x: [u16; 0] }` would have an alignment of 2
bytes by default. ([This matches the behavior in gcc and
clang](https://godbolt.org/z/5w0gkq).)

**Structs with fields of zero-size.** If a `#[repr(C)]` struct
containing a field of zero-size, that field does not occupy space in
the struct; it can affect the offsets of subsequent fields if it
induces padding due to the alignment on its type. ([This matches the
behavior in gcc and clang](https://godbolt.org/z/5w0gkq).)

**C++ compatibility hazard.** As noted above when discussing structs
with no fields, C++ treats empty structs like `struct Foo { }`
differently from C and Rust. This can introduce subtle compatibility
hazards. If you have an empty struct in your C++ code and you make the
"naive" translation into Rust, even tagging with `#[repr(C)]` will not
produce layout- or ABI-compatible results.

### Fixed alignment

The `#[repr(align(N))]` attribute may be used to raise the alignment
of a struct, as described in [The Rust Reference][TRR-align].

[TRR-align]: https://doc.rust-lang.org/stable/reference/type-layout.html#the-align-representation

### Packed layout

The `#[repr(packed(N))]` attribute may be used to impose a maximum
limit on the alignments for individual fields. It is most commonly
used with an alignment of 1, which makes the struct as small as
possible. For example, in a `#[repr(packed(2))]` struct, a `u8` or
`u16` would be aligned at 1- or 2-bytes respectively (as normal), but
a `u32` would be aligned at only 2 bytes instead of 4. In the absence
of an explicit `#[repr(align)]` directive, `#[repr(packed(N))]` also
sets the alignment for the struct as a whole to N bytes.

The resulting fields may not fall at properly aligned boundaries in
memory. This makes it unsafe to create a Rust reference (`&T` or `&mut
T`) to those fields, as the compiler requires that all reference
values must always be aligned (so that it can use more efficient
load/store instructions at runtime). See the [Rust reference for more
details][TRR-packed].

[TRR-packed]: https://doc.rust-lang.org/stable/reference/type-layout.html#the-packed-representation

<a name="fnabi"> </a>

### Function call ABI compatibility

In general, when invoking functions that use the C ABI, `#[repr(C)]`
structs are guaranteed to be passed in the same way as their
corresponding C counterpart (presuming one exists). `#[repr(Rust)]`
structs have no such guarantee. This means that if you have an `extern
"C"` function, you cannot pass a `#[repr(Rust)]` struct as one of its
arguments. Instead, one would typically pass `#[repr(C)]` structs (or
possibly pointers to Rust-structs, if those structs are opaque on the
other side, or the callee is defined in Rust).

However, there is a subtle point about C ABIs: in some C ABIs, passing
a struct with one field of type `T` as an argument is **not**
equivalent to just passing a value of type `T`. So e.g. if you have a
C function that is defined to take a `uint32_t`:

```C
void some_function(uint32_t value) { .. }
```

It is **incorrect** to pass in a struct as that value, even if that
struct is `#[repr(C)`] and has only one field:

```rust,ignore
#[repr(C)]
struct Foo { x: u32 }

extern "C" some_function(Foo);

some_function(Foo { x: 22 }); // Bad!
```

Instead, you should declare the struct with `#[repr(transparent)]`,
which specifies that `Foo` should use the ABI rules for its field
type, `u32`. This is useful when using "wrapper structs" in Rust to
give stronger typing guarantees.

`#[repr(transparent)]` can only be applied to structs with a single
field whose type `T` has non-zero size, along with some number of
other fields whose types are all zero-sized (typically
`std::marker::PhantomData` fields). The struct then takes on the "ABI
behavior" of the type `T` that has non-zero size.

(Note further that the Rust ABI is undefined and theoretically may
vary from compiler revision to compiler revision.)

## Unresolved question: Guaranteeing compatible layouts?

One key unresolved question was whether we would want to guarantee
that two `#[repr(Rust)]` structs whose fields have the same types are
laid out in a "compatible" way, such that one could be transmuted to
the other. @rkruppe laid out a [number of
examples](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-419956939)
where this might be a reasonable thing to expect. As currently
written, and in an effort to be conservative, we make no such
guarantee, though we do not firmly rule out doing such a thing in the future.  

It seems like it may well be desirable to -- at minimum -- guarantee
that `#[repr(Rust)]` layout is "some deterministic function of the
struct declaration and the monomorphized types of its fields".  Note
that it is not sufficient to consider the monomorphized type of a
struct's fields: due to unsizing coercions, it matters whether the
struct is declared in a generic way or not, since the "unsized" field
must presently be [laid out last in the
structure](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/12#issuecomment-417843595). (Note
that tuples are always coercible (see [#42877] for more information),
and are always declared as generics.) This implies that our
"deterministic function" also takes as input the form in which the
fields are declared in the struct.

However, that rule is not true today. For example, the compiler
includes an option (called "optimization fuel") that will enable us to
alter the layout of only the "first N" structs declared in the
source. When one is accidentally relying on the layout of a structure,
this can be used to track down the struct that is causing the problem.

[#42877]: https://github.com/rust-lang/rust/issues/42877
[pg-unsized-tuple]: https://play.rust-lang.org/?gist=46399bb68ac685f23beffefc014203ce&version=nightly&mode=debug&edition=2015

There are also benefits to having fewer guarantees. For example:

- Code hardening tools can be used to randomize the layout of individual structs.
- Profile-guided optimization might analyze how instances of a
particular struct are used and tweak the layout (e.g., to insert
padding and reduce false sharing).
    - However, there aren't many tools that do this sort of thing
([1](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420650851),
[2](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420681763)). Moreover,
it would probably be better for the tools to merely recommend
annotations that could be added
([1](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420077105),
[2](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420077105)),
such that the knowledge of the improved layouts can be recorded in the
source.

As a more declarative alternative, @alercah [proposed a possible
extension](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/12#issuecomment-420165155)
that would permit one to declare that the layout of two structs or
types are compatible (e.g., `#[repr(as(Foo))] struct Bar { .. }`),
thus permitting safe transmutes (and also ABI compatibility).  One
might also use some weaker form of `#[repr(C)]` to specify a "more
deterministic" layout. These areas need future exploration.

## Counteropinions and other notes

@joshtripplet [argued against reordering struct
fields](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-417953576),
suggesting instead it would be better if users reordering fields
themselves. However, there are a number of downsides to such a
proposal (and -- further -- it does not match our existing behavior):

- In a generic struct, the [best ordering of fields may not be known
  ahead of
  time](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420659840),
  so the user cannot do it manually.
- If layout is defined, and a library exposes a struct with all public
  fields, then clients may be more likely to assume that the layout of
  that struct is stable. If they were to write unsafe code that relied
  on this assumption, that would break if fields were reordered. But
  libraries may well expect the freedom to reorder fields. This case
  is weakened because of the requirement to write unsafe code (after
  all, one can always write unsafe code that relies on virtually any
  implementation detail); if we were to permit **safe** casts that
  rely on the layout, then reordering fields would clearly be a
  breaking change (see also [this
  comment](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420117856)
  and [this
  thread](https://github.com/rust-rfcs/unsafe-code-guidelines/pull/31#discussion_r224955817)).
- Many people would prefer the name ordering to be chosen for
  "readability" and not optimal layout.

[1-ZST]: ../glossary.md#zero-sized-type--zst

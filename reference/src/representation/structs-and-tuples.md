# Representation of structs and tuples

**Disclaimer:** This chapter represents the consensus from issues
[#11] and [#12].  The statements in here are not (yet) "guaranteed"
not to change.

[#11]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11
[#12]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/12

## Tuple types

In general, an anonymous tuple type `(T1..Tn)` of arity N is laid out
"as if" there were a corresponding tuple struct declared in libcore:

```rust
#[repr(Rust)]
struct TupleN<P1..Pn>(P1..Pn);
```

In this case, `(T1..Tn)` would be compatible with `TupleN<T1..Tn>`.
As discussed below, this generally means that the compiler is **free
to re-order field layout** as it wishes. Thus, if you would like a
guaranteed layout from a tuple, you are generally advised to create a
named struct with a `#[repr(C)]` annotation (see [the section on
structs for more details](#structs)).

There is one exception: if all N fields of the tuple are of the same
type `T` (with lifetime erased), then the tuple is guaranteed to be
laid out as the fixed-length array type `[T; N]` (with the numbered
tuple fields placed in the corresponding indices as expected). This
permits such tuples to be transmuted and then indexed using an integer
index.[^exception]

[^exception]: [Proposed in this comment](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/12#issuecomment-417680324).

### Other notes on tuples

Some related discussion:

- [RFC #1582](https://github.com/rust-lang/rfcs/pull/1582) proposed
  that tuple structs should have a "nested representation", where
  e.g. `(T1, T2, T3)` would in fact be laid out as `(T1, (T2,
  T3))`. The purpose of this was to permit variadic matching and so
  forth against some suffix of the struct. This RFC was not accepted,
  however. This lay out requires extra padding and seems somewhat
  surprising: it means that the layout of tuples and tuple structs
  would diverge significantly from structs with named fields.  This
  proposal is also incompatible with the guaranteed array layout
  described above.

<a name="structs"></a>

## Struct types

Structs come in two principle varieties:

```rust
// Structs with named fields
struct Foo { f1: T1, .., fn: Tn }

// Tuple structs
struct Foo(T1, .., Tn)
```

In general, tuple structs can be understood as equivalent to named
structs declared with the same order; therefore, the two declarations
of `Foo` in the previous example are treated equivalently.

Structs can have various `#[repr]` flags that influence their layout:

- `#[repr(Rust)]` -- the default.
- `#[repr(C)]` -- request C compatibility
- `#[repr(align(N))]` -- specify the alignment
- `#[repr(packed)]` -- request packed layout where fields are not internally aligned
- `#[repr(transparent)]` -- request that a single-field struct be
  treated "as if" it were an instance of its field type when passed as
  an argument

### Default layout ("repr rust")

The default layout of structs is undefined and subject to change
between compiler revisions. We further do not guarantee that two
structs with different names (but the same field types) will be laid
out in the same way (for example, the hypothetical struct representing
tuples ). Finally, the presence or absence of generics can make a
difference (e.g., `struct Foo { x: u16, y: u32 }` and `struct Foo<T> {
x: u16, y: T }` where `T = u32` are not guaranteed to be identical),
owing to the possibility of unsizing coercions.

**Compiler's current behavior.** As of the time of this writing, the
compiler will reorder struct fields to minimize the overall size of
the struct (and in particular to eliminate padding due to alignment
restrictions). The final field, however, is not reordered if an
unsizing coercion may be applied.

### C-compatible layout ("repr C")

For structs tagged `#[repr(C)]`, the compiler will apply a C-like
layout scheme (see section 6.7.2.1 of the [C17 specification][C17] for
a detailed write-up):

[C17]: http://www.open-std.org/jtc1/sc22/wg14/www/abq/c17_updated_proposed_fdis.pdf

- Field order is preserved.
- The first field begins at offset 0.
- Assuming the struct is not packed, each field's offset is aligned to
  the natural alignment for that field's type, possibly creating
  unused padding bits.
- The total size of the struct is rounded up to its overall alignemnt.  

The intention is that if one has a set of C struct declarations and a
corresponding set of Rust struct declarations, all of which are tagged
with `#[repr(C)]`, then the layout of those structs will all be
identical. Note that this setup implies that none of the structs in
question can contain any `#[repr(Rust)]` structs (or Rust tuples), as
those would have no corresponding C struct declaration -- as
`#[repr(Rust)]` types have undefined layout, you cannot safely declare
their layout in a C program.

See also the notes on ABI compatibility under the section on `#[repr(transparent)]`.

### Fixed alignment

The `#[repr(align(N))]` attribute may be used to raise the alignment
of a struct, as described in [The Rust Reference][TRR-align].

[TRR-align]: (https://doc.rust-lang.org/stable/reference/type-layout.html#the-align-representation).

### Packed layout

The `#[repr(packed)]` attribute may be used to remove all padding from
the struct. The resulting fields may not fall at properly aligned
boundaries in memory. This makes it unsafe to create a Rust reference
(`&T` or `&mut T`) to those fields, as the compiler requires that all
reference values must always be aligned (so that it can use more
efficient load/store instructions at runtime). See the [Rust reference
for more details][TRR-packed].

[TRR-packed]: https://doc.rust-lang.org/stable/reference/type-layout.html#the-packed-representation

### ABI Compatibility

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

```rust
#[repr(C)]
struct Foo { x: u32 }

extern "C" some_function(Foo);

some_function(Foo { x: 22 }); // Bad!
```

Instead, you should declare the struct with `#[repr(transparent)]`,
which specifies that `Foo` should use the ABI rules for its field
type, `u32`. This is useful when using "wrapper structs" in Rust to
give stronger typing guarantees.

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

There are also benefits also to having fewer guarantees. For example:

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

@joshtrippler [argued against reordering struct
fields](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-417953576),
suggesting instead it would be better if users reordering fields
themselves. However, there are a number of downsides to such a
proposal (and -- further -- it does not match our existing behavior):

- In a generic struct, the [best ordering of fields may not be known
  ahead of
  time](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420659840),
  so the user cannot do it manually.
- If layout is defined, then it becomes part of your API, such taht
  reordering fields is a breaking change for your clients (if we
  consider unsafe code that may rely on the layout, then this applies
  [even to structs with named
  fields](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11#issuecomment-420117856).
- Many people would prefer the name ordering to be chosen for
  "readability" and not optimal layout.

## Footnotes

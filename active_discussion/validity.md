# Data type validity requirements

This discussion is meant to focus on the question: Which invariants derived from
types are there that the compiler expects to be *always* maintained, and
(equivalently) that unsafe code must *always* uphold.  This is what is called
"validity invariant" in
[Ralf's blog post](https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html),
but we might also decide to change that name.

### Interactions and constraints

Choices of invariants interact, in particular, with layout optimizations: For
example, the fact that `Option<&T>` is pointer-sized relies on the fact that the
validity invariant for `&T` rules out `0x0`, and hence we can use that value as
signaling the `None` case.

Moreover, the invariants are constrained by attributes that we emit when
generating LLVM IR.  For example, we emit `aligned` attributes pretty much any
time we can, which means it is probably a good idea to say that valid references
must be aligned.

### Extent of "always"

One point we will have to figure out is what exactly "always" means.  Thinking
in terms of a semantics for MIR, data most probably needs to be valid any time
it is copied, which primarily happens when executing assignment statements (the
other cases are passing of function arguments and return values).  However, it
is less clear whether merely creating a place without accessing the data inside
(such as in `&*x`) should require the data to be valid.

### Possible bit patterns

The validity invariant of a type is, basically, a set of bit patterns that is
allowed to occur at that type.  ("Basically" because the invariant may also be
allowed to depend on memory.)  To discuss this properly, we need to first agree
on what "bit patterns" even are.  It is certainly not enough to just consider
sequences of 0 and 1, because we also need to take uninitialized data into
account.  For the purpose of this discussion, I think it is sufficient to
consider every bit as being either 0, 1 or uninitialized.
[That is not always sufficient](https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html),
but I think we can mostly ignore the extra complications introduced by pointer
values.

## Goals

* For every primitive type, determine which assumptions (if any) the compiler
  makes about values *not* occurring at that type (serving as a lower bound for
  what to declare invalid), and determine which popular patterns in unsafe code
  might create "interesting" values of this type that safe code cannot create on
  its own (serving as an upper bound for how much we want to declare invalid).
  Both of these bounds are soft, but informative.
* Based on that, map out a design space of invariants that seem reasonable.
* Determine when exactly the validity invariant is assumed to hold.

## Active threads

To start, we will create threads for each major category of types.

* Integers and floating point types

  * Do we allow values that contain uninitialized bits?  If yes, what are the
    rules for arithmetic and logical operations involving uninitialized bits,
    e.g. in cases like `x * 0`?

* Raw pointers
  * Do we allow values that contain uninitialized bits?
  * Are there any requirements on the metadata?

* References
  * Presumably, references must be non-NULL.
  * They probably also must be aligned, but is that required every time a
    reference is taken?  Also see the [ongoing discussion in RFC 2582][RFC2582].
  * Can there ever be uninitialized bits in a reference?
  * Do they have to be dereferencable?  What exactly does that even mean?
  * Does `&[mut] T` have to point to data that is valid at `T`?  This interacts
    with the question of whether `&*x` is allowed when `x` is a well-aligned
    non-null dereferencable pointer that points to invalid data.
  * Out of scope: aliasing rules

* Function pointers
  * Presumably, these must be non-NULL.  Anything else?  Can there ever be
    uninitialized bits?

* Unions
  * Do we make any restrictions here, or are unions just "bags of bits" that may
    contain anything?  That would mean we can do no layout optimizations.

* Enums
  * Is there anything to say besides: The discriminant must be valid, and all
    fields of the active variant must be valid at their respective types?
  * The padding between fields can be anything, including uninitialized.

* Structs, tuples, arrays and all other aggregates (closures, ...)
  * Is there anything to say besides: All fields must be valid at their
    respective types?
  * The padding between fields can be anything, including uninitialized.  It was
  * [recently determined][generators-maybe-uninit] that generators behave
  * different from other aggregates here.  Are we okay with that?  Should we push
  * for generator fields to reflect this in their types?

[RFC2582]: https://github.com/rust-lang/rfcs/pull/2582
[generators-maybe-uninit]: https://github.com/rust-lang/rust/pull/56100


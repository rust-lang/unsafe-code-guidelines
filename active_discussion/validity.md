# Data type validity requirements

This discussion is meant to focus on the question: Which invariants derived from
types are there that the compiler expects to be *always* maintained, and
(equivalently) that unsafe code must *always* uphold (or else cause undefined
behavior)?  This is what is called "validity invariant" in
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

Finally, another consideration to take into account is that ruling out certain
behavior can be great for bug finding.  For example, if arithmetic overflow is
defined to have two's-complement-behavior, then bug finding tools can no longer
use overflow as an indication of a software bug.  (This is a real problem with
unsigned integer arithmetic in C/C++.)

### Possible bit patterns

The validity invariant of a type is, basically, a set of bit patterns that is
allowed to occur at that type.  ("Basically" because the invariant may also be
allowed to depend on memory.)  To discuss this properly, we need to first agree
on what "bit patterns" even are.  It is not enough to just consider sequences of
0 and 1, because we also need to take uninitialized data into account.  For the
purpose of this discussion, I think it is sufficient to consider every bit as
being either 0, 1 or uninitialized.
[That is not always sufficient](https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html),
but I think we can mostly ignore the extra complications introduced by pointer
values.

In terms of comparing with C, the "uninitialized" bit corresponds to what C
calls "indeterminate" data.  In particular, it is allowed to be a "trap
representation".  Also, observing the same indeterminate data multiple times is
allowed to yield different results.  That's why I am proposing we treat it as a
third state a bit can be in.

In terms of LLVM, the "uninitialized" bit corresponds to `poison`.  It is *not*
the same as `undef`!  See
[this paper](https://www.cs.utah.edu/~regehr/papers/undef-pldi17.pdf) for some
more material on the topic.

### Extent of "always"

One point we will have to figure out is what exactly "always" means.  Thinking
in terms of a semantics for MIR, data most probably needs to be valid any time
it is copied, which primarily happens when executing assignment statements (the
other cases are passing of function arguments and return values).  However, it
is less clear whether merely creating a place without accessing the data inside
(such as in `&*x`) should require the data to be valid.

The entire discussion here is only about validity invariants that have to hold
when the compiler considers a variable initialized.  For example, `let b: bool;`
is completely okay to not be initialized because the compiler knows about that;
`let b: bool = mem::uninitialized();` however copies uninitialized data at type
`bool` and hence violates `bool`'s validity invariant.

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
    e.g. in cases like `x * 0`?  There is also some interaction with bug finding
    here: tools can only flag uninitialized data at integer type as a bug if we
    do not allow that to happen in unsafe code.

* Raw pointers
  * Do we allow values that contain uninitialized bits?
  * Are there any requirements on the metadata?

* References

    I propose splitting this discussion into three pieces.

  * Bit-level properties
    * Presumably, references must be non-NULL.
    * They probably also must be aligned, but is that required every time a
      reference is taken?  Also see the [ongoing discussion in RFC 2582][RFC2582].
    * Can there ever be uninitialized bits in a reference?
  * Memory-related properties
    * Do references have to be dereferencable?  What exactly does that even
      mean?  We have a design choice here to make this *not* part of validity,
      but part of the aliasing model instead (and in fact, that is what miri
      currently implements).  In terms of Stacked Borrows, the operation that
      asserts references being dereferencable is `Retag`, and not the validity
      check happening at the assignment.  That helps to keep validity
      independent of the state of memory.
    * Does `&[mut] T` have to point to data that is valid at `T`?  This interacts
      with the question of whether `&*x` is allowed when `x` is a well-aligned
      non-null dereferencable pointer that points to invalid data.
  * Size-related properties
    * For references to unsized types, does validity require the metadata to
      make sense?  Valid metadata is required to even define a notion of
      "dereferencable", because we have to specify the extent of memory that is
      dereferencable (i.e., we have to specify how many bytes are
      dereferencable).

      On the other hand, this makes validity depend on memory, at least for
      vtables.  However, vtables are somewhat special memory: They are never
      deallocated and never mutated.  So while determining the size depends on
      memory, we know for sure that the computed size cannot ever change for a
      given reference.

      All of this gets much, much more complicated with custom DSTs.  For
      example, for a C-style string pointer, does validity require there to be a
      0-terminator?  Should checking for validity, and/or retagging, really
      execute arbitrary user-defined code to determine the extent of memory
      covered by this reference?

  * Out of scope: aliasing rules

* Function pointers
  * Presumably, these must be non-NULL.  Anything else?  Can there ever be
    uninitialized bits?

* Booleans
  * Is there anything to say besides: A `bool` must be `0x0` or `0x1`?  Do we
    allow the remaining bits to be uninitialized?

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
    [recently determined][generators-maybe-uninit] that generators behave
    different from other aggregates here.  Are we okay with that?  Should we push
    for generator fields to reflect this in their types?

* `ManuallyDrop`
  * `ManuallyDrop` might be special in terms of the validity invariant.
    Probably it requires its data to be bitstring-valid, but does a
    `ManuallyDrop<&T>` have to be dereferencable?

[RFC2582]: https://github.com/rust-lang/rfcs/pull/2582
[generators-maybe-uninit]: https://github.com/rust-lang/rust/pull/56100


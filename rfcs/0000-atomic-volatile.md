- Feature Name: `atomic_volatile`
- Start Date: 2019-10-15
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Introduce a set of `core::volatile::VolatileXyz` structs, roughly modeled after
the existing `core::sync::atomic::AtomicXyz` APIs, which expose volatile loads
and stores of machine-native width with atomic semantics. Recommend that these
operations be preferred to `ptr::[read|write]_volatile` and the corresponding
methods of pointer types, as they have stronger semantics that are almost always
closer to user intent.


# Motivation
[motivation]: #motivation

Volatile operations are meant to be an escape hatch that allows a Rust
programmer to invoke hardware-level memory access semantics that are not
properly accounted for by the Rust Abstract Machine. They work by triggering the
largely unoptimized generation of a matching stream of hardware load and store
instructions in the output machine code.

Unfortunately, the volatile operations that are currently exposed by Rust, which
map into LLVM's non-atomic volatile operations, unnecessarily differ from the
memory access semantics of mainstream hardware in two major ways:

1. Concurrent use of volatile memory operations on a given memory location is
   considered to be Undefined Behavior, and may therefore result in unintended
   compilation output if detected by the optimizer.
2. Using an overly wide volatile load or store operation which cannot be carried
   out by a single hardware load and store instruction will not result in a
   compilation error, but in the silent emission of multiple hardware load or
   store instructions, which might be a logic error in some programs.

By implementing support for LLVM's atomic volatile operations, and encouraging
their use on every hardware that supports them, we eliminate these divergences
from hardware behavior and therefore bring volatile operations closer to the
"defer to hardware memory model" semantics that programmers expect them to have.
This reduces the odd of mistake in programs operating outside of the regular
Rust Abstract Machine semantics, which are notoriously hard to get right.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Volatile primer
[volatile-primer]: #volatile-primer

The Rust compiler generally assumes that the program that it is building is
living in a fully isolated memory space, where the contents of memory can only
change if some direct action from the program (including FFI or atomic memory
operations) allows it to change.

It leverages this knowledge to transform said program's memory access patterns
for performance optimization purposes, under the assumption that said
transformations will not have any externally observable effect other than
speeding up the program.

Examples of such transformations include:

- caching data from RAM into CPU registers,
- spilling CPU registers into accessible RAM locations,
- eliminating unused loads and unobservable stores,
- merging neighboring stores with each other,
- only updating the part of an over-written struct that has actually changed.

Although these optimizations are most of the time correct and useful, there are
some situations where they are inappropriate, in areas such as:

- [Memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O), a common
  low-level communication protocol between CPUs and peripherals, where hardware
  registers masquerading as memory can be used to program peripherals by
  accessing said registers in very specific load/store patterns.
- [Shared-memory IPC](https://en.wikipedia.org/wiki/Shared_memory), a form of
  inter-process communication where two programs can communicate via a common
  memory block, which means that stores are externally observable and loads are
  not guaranteed to return consistent results. This is not to be confused with
  shared-memory concurrency, which refers to sharing of memory between multiple
  threads running within the same process.
- Advanced uses of [virtual memory](https://en.wikipedia.org/wiki/Virtual_memory)
  where the mere action of reading from or writing to memory may trigger
  execution of arbitrary code by the operating system. Note that even when using
  volatile accesses, [some sanity restrictions](https://llvm.org/docs/LangRef.html#volatile-memory-accesses)
  are imposed by LLVM here to allow optimization of surrounding code.
- [Cryptography](https://en.wikipedia.org/wiki/Cryptography), where it is
  extremely important to ensure that sensitive information is erased after use,
  and is not leaked via indirect means such as recognizable scaling patterns in
  the time taken by a system to process attacker-crafted inputs.

In all those circumstances, though for different reasons, it may be important to
guarantee that memory loads and stores specified in the program do occur at the
hardware level, because they have externally observable side-effects outside of
the Rust program being optimized, and may be subjected to unpredictable
side-effects from the outside world.

When that happens, it is useful to be able to assert precise manual control on
the memory accesses that are carried out by a Rust program in a certain memory
region. This is the purpose of _volatile memory operations_, which allow a Rust
programmers to generate a carefully controlled stream of hardware memory load
and store instructions, which is guaranteed to be left untouched by the Rust
compiler's optimizer even though surrounding Rust code will continue to be
optimized as usual.

## API guide
[api-guide]: #api-guide

### Basic usage
[basic-usage]: #basic-usage

Volatile memory operations are exposed in the `std::volatile` module of the Rust
standard library, or alternatively in the `core::volatile` module for the
purpose of writing `#![no_std]` applications. They are interfaced through
fixed-size data wrappers that are somewhat reminiscent of the API used for
atomic operations in `std::sync::atomic`:

```rust
use std::ptr::NonNull;
use std::volatile::VolatileU8;

unsafe fn do_volatile_things(target: NonNull<VolatileU8>) -> u8 {
    target.store(42);
    target.load()
}
```

Notice that volatile types must be manipulated via pointers, instead of
Rust references. These unusual and unpleasant ergonomics are necessary in order
to achieve the desired semantics of manually controlling every access to the
target memory location, because the mere existence of a Rust reference pointing
to a memory region allows the Rust compiler to generate extra memory operations
targeting this region (be they prefetches, register spills, ...).

Because a Rust pointer is not subjected to borrow checking and has no obligation
of pointing towards a valid memory location, this means that using a Volatile
wrapper in any way is unsafe.

### Concurrency and ordering
[concurrency-and-ordering]: #concurrency-and-ordering

A second difference with familiar atomic operations is that volatile type
operations do not have an `Ordering` parameter. These operations are atomic in
the sense that it is safe to perform them concurrently from multiple threads on
a single target memory location, but their ordering guarantees in multi-threaded
environments are unlike that of any atomic operation ordering:

- Within a single thread, these operations are guaranteed to be _executed_ by
  the underlying CPU in program order...
- ...but the order in which these operations are subsequently _observed_ to
  occur by another thread performing volatile operations on the same memory
  location is fully unspecified, and may defy even `Ordering::Relaxed` logic.

For example, assuming that a thread A successively writes the values 24 and 42
to a memory location using volatile operations, other threads B and C repeatedly
reading the value of said memory location are allowed to observe these writes
without any commonly agreed upon coherence ordering:

    Thread B:             Thread C:
    24                    42
    42                    24

As a matter of fact, other threads are not even guaranteed to eventually observe
either of the writes carried out by thread A if no further sychronization action
is taken to guarantee their visibility.

Individual hardware targets supported by Rust may and often will provide
stronger ordering and visibility guarantees under the above intra-thread
execution ordering constraint. But a program relying on these hardware
guarantees will not be portable to all hardware targets supported by Rust.

Readers who would still feel the perverted temptation to use volatile operations
for synchronization in spite of the above warning should keep in mind that the
presence of volatile operations does not affect the optimization of surrounding
non-volatile operations in any significant ways. Said operations may still be
added, elided, split, narrowed, extended, or reordered as deemed useful by the
compiler's optimizer, including by moving them across neighboring volatile
operations.

### No-elision guarantee
[no-elision-guarantee]: #no-elision-guarantee

Finally, a third difference with atomic operations is that the Rust compiler is
not allowed to remove volatile operations as part of the code optimization
process. For example, a compiler may not leverage knowledge of hardware memory
load/store semantics to elide the volatile load of the above function...

```rust
unsafe fn do_volatile_things(target: NonNull<VolatileU8>) -> u8 {
    target.store(42);
    42
}
```

...nor is it allowed to elide memory accesses to a given location entirely, even
if the code optimizer somehow managed to prove that no other thread seems to
have access to the `VolatileU8` variable under consideration:

```rust
unsafe fn do_volatile_things(_target: NonNull<VolatileU8>) -> u8 {
    42
}
```

The fact that hardware loads and stores must be emitted even when the compiler's
optimizer can predict the results of loads or assert that stores will have no
effect on program execution is one of the most central characteristics of
volatile operations. It is what makes these operations suitable for "sensitive"
memory manipulations such as cryptographic secret erasure or memory-mapped I/O.

### Further guarantees
[further-guarantees]: #further-guarantees

Volatile operations also provide a few more guarantees that are easier to
explain, which complete their feature set for the intended purpose of providing
tight control on memory accesses:

- _As long as a memory location is only accessed via `NonNull<VolatileXyz>`_, it
  is guaranteed that the compiler will not insert any extra accesses to it
  beyond the loads and stores that are present in the program.
- Although their ordering guarantees are weaker than `Ordering::Relaxed`,
  volatile operations are still atomic, which means that they...
    * ...cannot be split (e.g. an `u64` load/store cannot be transformed into
      two consecutive `u32` loads/stores).
    * ...cannot be narrowed (e.g. an `u64` load/store cannot be transformed into
      an `u8` load/store, even if the optimizer believes that only a subset of
      the target data has changed)
    * ...cannot be extended to store data which would not be stored otherwise.
- Beyond atomic guarantees, volatile operations also guarantee that they cannot
  be extended to load data which would not be loaded otherwise.

Combined with the above, these guarantees mean that volatile memory operations
happen as specified in the program, in the order specified by the program, and
that with a bit of work it is also possible to guarantee that they are the only
memory operations affecting the target memory location.

## Comparison with `ptr::[read|write]_volatile()`
[volatilexyz-vs-ptrvolatile]: #volatilexyz-vs-ptrvolatile

Experienced Rust users may be familiar with the previously existing
`std::ptr::read_volatile()` and `std::ptr::write_volatile()` functions, or
equivalent methods of raw pointer objects, and wonder how the new facilities
provided by `std::volatile` compare to those methods.

The answer is that this new volatile data access API should be preferred to its
predecessor in almost every use case, as it improves upon it in several ways:

- Unlike its predecessor, this API does not specify concurrent volatile accesses
  to a region of memory to be undefined behavior. This undefined behavior made
  no sense to begin with, because...
    * The purpose of volatile memory accesses is to go as close to target
      hardware semantics as possible in a high-level programming language, and
      all targets supported by Rust have well-defined data race semantics.
    * No performance optimization potential is gained by having this undefined
      behavior, since volatile operations cannot be meaningfully optimized under
      the assumption that target memory is not concurrently read or modified.
- Unlike its predecessor, this API is only available for data types which have
  a supported machine load/store operation at the hardware level. Therefore, one
  no longer needs to worry about the possibility of the compiler silently
  splitting a Rust-level volatile data access into multiple hardware-level
  memory operations. In the same manner as with atomics, if a `Volatile` wrapper
  type is provided by Rust, the underlying hardware is guaranteed to support
  memory operations of that width.
- Unlike its predecessor, this API sets a clear strong typing barrier to prevent
  developers from mixing volatile and non-volatile memory accesses, which is
  almost always a mistake.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Semantics
[semantics]: #semantics

The fundamental purpose of volatile operations, in a system programming language
like Rust, is to allow a developer to locally escape the Rust Abstract Machine's
weak memory semantics and defer to the hardware's memory semantics instead,
without resorting to the full complexity, overhead, and non-portability of
assembly (inline or otherwise).

Rust's current volatile operations, which defer to LLVM's non-atomic volatile
operations, do not achieve this goal very well because:

1. Their data race semantics do not match the data race semantics of the
   hardware which volatile is supposed to defer to, and as a result are
   unnecessarily surprising and unsafe. No useful optimization opportunities are
   opened by this undefined behavior, since volatile operations cannot be
   meaningfully optimized.
2. The absence of a hard guarantee that each volatile load or store will
   translate into exactly one load or store at the hardware level is needlessly
   tracherous in scenarios where memory access patterns must be very precisely
   controlled, such as memory-mapped I/O.

Both of these problems can be resolved by using atomic volatile operations
instead of non-atomic volatile operations. However, in doing so, we must be very
careful to stick with the subset of atomic operations that is supported by all
Rust hardware targets.

This means that we should not expose anything more than loads or stores in our
standard subset, as some hardware architectures which are either very old or
lack multiprocessing support do not support more than that.

More surprisingly, maximizing hardware support also means that we cannot rely on
`Ordering::Relaxed` as a minimal level of atomic sanity, because this ordering
assumes a global order of operations targeting a single memory location across
all threads in the program, and this property is not provided by native
load/store operations on some targets without global cache coherence such as GPU
architectures (NVPTX, AMD GCN, ...).

As a result, we need to go deeper and expose an even weaker form of atomicity,
which is more akin to LLVM's `unordered` atomic ordering semantics, but better
specified in the context of atomic volatile operations because we provide very
strong control on the sequence of emitted hardware memory accesses, and thus
developers can refer to their hardware's memory model for precise semantics.

One design goal of the proposed semantics is that it should be possible to
implement `VolatileXyz::[load|store]` by compiling it down to LLVM unordered
atomic volatile load/store operations of the same width, without specifying it
in terms of "whatever LLVM unordered does" as that would make the life of
alternate rustc backends like cranelift harder.

## API design
[api-design]: #api-design

Switching to LLVM's atomic volatile accesses without also changing the API of
`ptr::[read|write]_volatile` would unfortunately not resolve the memory access
tearing problem very well, because although oversized volatile accesses would
not compile anymore, this fact would only be "reported" to the user via an LLVM
crash. This is not a nice user experience, which is why volatile wrapper types
are proposed instead.

Their design borrows from that of Rust's existing atomic types, which is only
appropriate since they do expose atomic operations. The current proposal would
be to fill the newly built `std::volatile` module with the following entities
(some of which may not be available on a given platform, we will come back to
this point in a moment):

- `VolatileBool`
- `VolatileI8`
- `VolatileI16`
- `VolatileI32`
- `VolatileI64`
- `VolatileIsize`
- `VolatilePtr`
- `VolatileU8`
- `VolatileU16`
- `VolatileU32`
- `VolatileU64`
- `VolatileUsize`

Unlike `AtomicXyz` types, these volatile types would be restricted to load and
store operations with sub-`Relaxed` concurrent ordering guarantees (see above),
and their API would be based on raw pointers instead of references. That is
because the existence of a Rust reference to a memory location currently implies
a `dereferenceable` annotation on the LLVM side, which in turn is fundamentally
at odds with the precise control of hardware load and store generation that is
required by volatile access use case.

To give a more concrete example, here is what the API of `VolatileBool` would
look like on a platform with support for this type:

```rust
#![feature(arbitrary_self_types)]
use std::sync::atomic::Ordering;


#[repr(transparent)]
struct VolatileBool(bool);

impl VolatileBool {
    /// Creates a new VolatileBool pointer
    ///
    /// This is safe as creating a pointer is considered safe in Rust and
    /// volatile adds no safety invariant to the input pointer.
    ///
    pub const fn new(v: NonNull<bool>) -> NonNull<Self> { /* ... */ }

    /// Load the target bool value in an atomic and volatile way
    ///
    /// # Safety
    ///
    /// The `self` pointer must be well-aligned and point to a valid memory
    /// location containing a valid `bool` value.
    ///
    pub unsafe fn load(self: NonNull<Self>) -> bool { /* ... */ }

    /// Store a bool into the target in an atomic and volatile way
    ///
    /// # Safety
    ///
    /// The `self` pointer must be well-aligned and point to a valid memory
    /// location containing a valid `bool` value.
    ///
    pub unsafe fn store(self: NonNull<Self>, val: bool) { /* ... */ }
}
```

## Availability
[availability]: #availability

Like `std::sync::atomic::AtomicXyz` APIs, the `std::volatile::VolatileXyz` APIs
are not guaranteed to be supported on every platform. As with atomic APIs, the
classic reason why a `VolatileXyz` wrapper would be missing is that the target
platform does not have load and store instructions of the corresponding width.

However, the `VolatileXyz` APIs could be made available on some platforms where
atomic operations cannot be, because they require much weaker concurrent
ordering semantics from hardware in their quest to be usable in every place
where native loads and store instructions can be used.

Therefore, we propose to provide a new platform support query,
`#[cfg(target_has_volatile = N)]`, which works in essentially the same fashion
as `#[cfg(target_has_atomic_load_store = N)]` and may initially be implemented
as a synonym of the latter, but can be eventually extended to provide volatile
load/store support on platforms where `Relaxed` atomic loads and stores are not
available or more complex than basic loads and stores.


# Drawbacks
[drawbacks]: #drawbacks

## API duplication
[api-duplication]: #api-duplication

Keeping around two different and subtly incompatible ways to do almost the same
thing, in the form of these new wrappers and the old volatile read/write methods
of pointer types, is unsatisfying. It could be argued that the issues with the
existing volatile operations, while real, do not warrant the full complexity of
this RFC.

Note that duplication of implementation should not be necessary as
`ptr::[read|write]_volatile` should be implementable on top of
`VolatileXyz::[load|store]`, in the spirit of what LLVM's element-wise atomic
memcpy already does.

## Unordered semantics
[unordered-semantics]: #unordered-semantics

This RFC also introduces new semantics for atomic operations, morally equivalent
to LLVM's unordered atomic volatile loads and stores. This is risky because
LLVM's unordered atomic semantics have not been sufficiently vetted for
soundness by the language memory model community. In the RFC author's opinion,
the risk is worthwhile because...

- The alternative of non-atomic volatile is worse. It makes data races undefined
  behavior in violation of any known hardware memory model, and therefore it
  takes volatile away from its "defer to hardware memory model" intent.
- The alternative of `Relaxed` atomic volatile is also worse, it makes the new
  API for volatile operations unusable on hardware without global cache
  coherence, such as GPUs.
- Although unordered atomics are a bit of a minefield, unordered volatile is
  safer to the point of being reasonable, because volatile provides so many
  guarantees on its own that adding unordered semantics on top of it should have
  the sole effect of preventing tearing and making data races well-defined.

## Pointer-based API
[pointer-based-api]: #pointer-based-api

Because rustc treats references as `dereferenceable` at the LLVM layer, which is
incompatible with the degree of memory access control that volatile access
users typically demand, this RFC is forced to design an API around raw pointers.

Using raw pointers means bypassing the borrow checker, which increases the risk
of memory unsafety. It also makes for a less familiar API design that users will
have a harder time getting to grips with. In this sense, maybe it would just be
better to resolve the `dereferenceable` issue first (allowing some sort of
non-`dereferenceable` reference which remains subjected to all other normal
guarantees of references), and then redesign this API in terms of whatever the
chosen solution ends up being.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This effort seems worthwhile because it eliminates two well-known and
unnecessary footguns of non-atomic volatile operations, namely data races being
undefined behavior and tearing not being a compilation error.

But although this feature may seem simple, its design space is actually
remarquably large, and a great many alternatives were considered before reaching
the current design proposal. Here are some design knobs that were explored:

## Extending `AtomicXyz`
[extending-atomicxyz]: #extending-atomicxyz

Atomic volatile operations could be made part of the Atomic wrapper types.
However, they cannot be just another atomic memory ordering, due to the fact
that atomic ops take an `&self`, which is against the spirit of volatile as it
is tagged with `dereferenceable` at the LLVM layer.

Instead, one would need to add more methods to atomic wrapper types, which take
a pointer as a self-parameter. This API inconsistency was felt to be
unnecessarily jarring to users.

In addition, one intent of this API is to be usable in places where even
`Relaxed` ordering is not usable, while punting on the stabilization of weaker
atomic orderings in the non-volatile case. This would mean adding one third
"tier" of hardware atomics support (beyond `target_has_atomic_load_store` and
`target_has_atomic`), which starts to feel a bit much.

Finally, it is extremely common to want either all operations on a memory
location to be volatile, or none of them. Providing separate wrapper types
helps enforce this very common usage pattern at the API level.

If a need for mixed volatile and non-volatile operations on a given memory
location ever emerges, we could envision providing a new `VolatileXyz` method
that casts from `NonNull<VolatileXyz>` to `&AtomicXyz`, with a clear warning in
its documentation that doing so voids the warranty of no out-of-thin-air memory
operations that `VolatileXyz` tries to hard to provide.

## Self-type or not self-type
[self-type]: #self-type

As currently designed, this RFC uses `arbitrary_self_types` to give method-like
semantics to a `NonNull` raw pointer. This seems necessary to get reasonable
ergonomics with an atomics-like wrapper type approach. However, it could also be
argued that a `VolatileXyz::store(ptr, data)` style of API would work
well enough, and avoid coupling with unstable features.

This would have the advantage of avoiding coupling this feature to another
unstable feature. But it would have the drawback of being very verbose. Typing
the same volatile type name over and over again in a complex transaction would
certainly obscure the intent of the code, whereas code that requires use of
volatile operations is often very subtle and benefits from being as readable as
it gets.

## `NonNull<T>` vs `*mut T` vs `*const T` vs other
[pointers]: #pointers

It is pretty clear that volatile operations cannot be expressed through `&self`
Rust references, as these provide the compiler with a licence to inject
arbitrary loads from the memory region (or even stores, for register spills, if
the region is determined to be unobservable to other threads). This would be
incompatible with the stated goal of precisely controlling memory accesses.

Currently, the only alternative to references in Rust is to use raw pointer
types. Rust has a number of these, here it is proposed to use `NonNull<T>`
pointer type because it encodes the non-nullness invariant of the API in code.
Although `NonNull<T>` is often less ergonomic to manipulate than `*mut T`, the
use of arbitrary self types make its ergonomics comparable in this case.

But it should be noted that overall, using raw pointers for this API feels
decidedly unsatisfactory overall. It feels like this use case could benefit from
a different kind of data accessor which encodes the fact that the data must be
live and valid, by correctly upholding normal borrow-checking rules.

In this way, most of the proposed API could become safe, and the only thing that
would remain unsafe would be the `_not_atomic()` operations, which more closely
reflects the reality of the situation.

One possible way to achieve this result would be to introduce a way to disable
the licence to insert arbitrary loads from Rust references that the compiler
normally has. For example, it has been proposed before that `&UnsafeCell<T>`
should not exhibit this behavior. This could be enough for `VolatileXyz`'s need
if it were extended to transparent newtypes of `UnsafeCell<Xyz>`.

## Hardware loads/store vs full atomics vocabulary
[atomics-vs-hardware]: #atomics-vs-hardware

The current version of this RFC proposes to restrict the `VolatileXyz` API to
atomic loads and stores with no concurrent ordering guarantees. It could
alternatively expose the full functionality of LLVM's atomic volatile operations
in the same manner as `AtomicXyz` does for non-volatile atomic operations.

There are three reasons why it was chosen not to do so:

- It goes against the typical intent of volatile, which is to exert tight
  control on the hardware load/store instructions that are generated.
- It would duplicate the `AtomicXyz` API to a large extent.
- There is no known use case for nontrivial atomic volatile operations that
  wouldn't be better served by adding volatile orderings to regular atomic
  operations, at a reduced API duplication cost.

Full discussion of this question, and of the proposed alternative, has therefore
been moved the "future possibilities" section.


# Prior art
[prior-art]: #prior-art

Atomic volatile accesses exist in C++11 and C11. They are respectively exposed
in those languages as [volatile overloads of `std::atomic` operations](https://en.cppreference.com/w/cpp/atomic/atomic/exchange) and [just making all atomic operations operate on
volatile objects](https://en.cppreference.com/w/c/atomic/atomic_load).

There is also a lot of prior art behind C's notion of volatile, which is closer
to Rust's current notion of volatile. That being said, most discussions of C/++
volatile semantics end up in complaints about how ill-specified, how easy it is
to get it wrong, how "contagious" it can get... so it isn't clear if it is a
very good role model to follow. Furthermore, Rust's notion of volatile differs
fundamentally from C's notion of volatile in that it is based on volatile
_operations_, not volatile _types_.

In the C++ community, there have been
[a series](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1152r0.html)
[of papers](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1152r2.html)
by JF Bastien to greatly reduce the scope of `volatile`, ideally until it
basically covers just loads and stores.

In the Rust community, we have a long history of discussing use cases that
require volatile semantics, from
[memory-mapped I/O](https://github.com/rust-lang/unsafe-code-guidelines/issues/33)
to [weird virtual memory](https://github.com/rust-lang/unsafe-code-guidelines/issues/28)
and [interaction with untrusted system processes](https://github.com/rust-lang/unsafe-code-guidelines/issues/152).

This last case, in particular, could be served through a combination of atomic
volatile with a clarification of LLVM's volatile semantics which would tune down
the amount of situations in which a volatile read from memory can be undefined
behavior.

There are plenty of crates trying to abstract volatile operations. Many are
believed to be unsound as they expose an `&self` to the sensitive memory
region, but `voladdress` is believed not to be affected by this problem.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

If we decide to implement this, implementation should be reasonably
straightforward and uneventful, as most of the groundwork has already been done
over the course of implementing `ptr::read_volatile()`, `ptr::write_volatile()`
and `std::sync::atomic` (to the best of the author's knowledge at least).

## Should shared-memory IPC always be volatile?
[should-shared-memory-be-volatile]: #should-shared-memory-be-volatile

There is [some ongoing discussion](https://github.com/rust-lang/unsafe-code-guidelines/issues/215)
in the Unsafe Code Guidelines group concerning whether a Rust implementation
should assume the existence of unknown threads of execution, even when no
concrete action has been taken to spawn such threads. This choice is a tradeoff
between Rust code performance and FFI ergonomics.

Depending on the outcome of this discussion, use cases such as shared-memory
interprocess communication, which do involve external threads which the Rust
implementation has no knowledge of, may or may not require systematic use of
volatile memory accesses.

- If we go in the "maximal Rust performance" direction, then every access to
  shared memory must be marked volatile because the Rust compiler is allowed to
  optimize it out if it is not subsequently used by Rust code (or if it can
  transform the Rust code to eliminate that use).
- If we go in the "maximal FFI ergonomics" direction, then volatile accesses are
  only needed when they are not coupled with atomics-based synchronization, as
  the mere presence of atomics acts as a trigger that disables the above
  optimizations.

## Safer self types
[safer-self-types]: #safer-self-types

This RFC would also benefit from a safer way to interact with volatile memory
regions than raw pointers, by providing a way to opt out of LLVM's
"dereferenceable" semantics without also having to opt out from all the
memory-safety guarantees provided by the Rust borrow checker.


# Future possibilities
[future-possibilities]: #future-possibilities

## Untrusted shared-memory IPC
[untrusted-shared-memory]: #untrusted-shared-memory

Although it performs a step in the right direction by strengthening the
definition of volatile accesses to reduce the amount of possible avenues for
undefined behavior, this RFC will no fully resolve the "untrusted shared memory"
use case, where Rust code is interacting with untrusted arbitrary code via a
shared memory region.

Doing so would also require work on clarifying LLVM semantics so that it is
absolutely clear that a malicious process cannot cause UB in another process by
by writing data in memory that's shared between the two, no matter if said
writes are non-atomic, non-volatile, or contains uninitialized data.

## Stronger orderings and RMW operations
[stronger-ordering-and-rmw]: #stronger-ordering-and-rmw

This RFC proposes to initially stabilize an absolute minimum of atomic volatile
operations, morally equivalent to LLVM's `unordered` atomic volatile loads and
stores. But atomic volatile operations can, in principle, support stronger
semantics, such as `Relaxed` and stronger orderings and read-modify-write
operations like `compare_and_swap`.

The reason why these operations are not made part of the `VolatileXyz` API is
that they interfere with one common intent behind volatile operations, which is
to exert tight control on the stream of hardware instructions that is emitted:

- `Relaxed` and stronger ordering may lead to the emission of different hardware
  load and store instructions, or to the emission of hardware memory barriers,
  in a target- and ABI-dependent fashion. As memory barriers can be coalesced,
  the details of which instructions will be used depend on surrounding code.
- `compare_and_swap` will lead to the generation a full loop (and thus break
  wait-freedom progress guarantees, which may be unexpected) on hardware based
  on the Load-Linked / Store-Conditional (LL/SC) synchronization formalism.
- Some atomic read-modify-write operations will lead to the generation a loop
  (and thus break wait-freedom progress guarantees, which may be unexpected) on
  hardware based on the Load-Linked / Store-Conditional (LL/SC) synchronization
  formalism. Which instructions will be subjected to this treatment depends on
  target CPU instruction set and whether the compiler backend will choose to use
  dedicated wait-free instructions over an LL/SC loop.

Clearly, the mental model for the process through which these instructions are
translated into hardware semantics is much more difficult to reason about, and
may not be stable across compiler versions and architecture/ABI revisions.

So far, the only use case that was found for these operations was to exert
stronger control over atomic operation emission, on a hypothetical backend
which would optimize atomics aggressively. This use case does not seem like it
would require the full constraints of the `VolatileXyz` API, and it could
therefore probably be turned into an extension of `std::sync::atomic::Ordering`.

One possibility would be to duplicate every ordering with a volatile variant
(e.g. `RelaxedVolatile`, `AcquireVolatile...`). Another possibility would be to
turn `std::sync::atomic::Ordering` into a bitfield, which would allow cleaner
syntax in the `SeqCst | Volatile` style.

## Deprecating `ptr::[read|write]_volatile`
[deprecating-ptr-volatile]: #deprecating-ptr-volatile

Finally, one could consider `ptr::[read|write]_volatile` and the corresponding
methods of pointer types as candidates for future deprecation, as they provide
less clear semantics than atomic volatile accesses (e.g. no guarantee of being
exempted from the definition of data races) for no clear benefit. As a more
backward-compatible alternative, one could also reimplement those methods using
a loop of atomic volatile operations of unspecified width.

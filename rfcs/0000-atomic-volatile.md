- Feature Name: `atomic_volatile`
- Start Date: 2019-10-15
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Introduce a set of `core::volatile::VolatileXyz` structs, modeled after the
existing `core::sync::atomic::AtomicXyz` API, which expose volatile loads and
stores of natively supported width with both atomic and non-atomic semantics.
Deprecate `ptr::[read|write]_volatile` and the corresponding methods of pointer
types, and recommend that they be replaced with `Relaxed` atomic volatile
operations on every platform that has support for them.


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
   store instructions, which might be a logic error in the users' program.

By implementing support for LLVM's atomic volatile operations, and encouraging
their use on every hardware that supports them, we eliminate these divergences
from hardware behavior and therefore bring volatile operations closer to the
"defer to hardware memory model" semantics that programmers expect them to have.
This reduces the odd of mistake in programs operating outside of the regular
Rust Abstract Machine semantics, which are notoriously hard to get right.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

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
  execution of arbitrary code by the operating system.
- [Cryptography](https://en.wikipedia.org/wiki/Cryptography), where it is
  extremely important to ensure that sensitive information is erased after use,
  and is not leaked via indirect means such as recognizable scaling patterns in
  the time taken by a system to process attacker-crafted inputs.

In all those circumstances, though for different reasons, it may be important to
guarantee that memory loads and stores do occur, because they have externally
observable side-effects outside of the Rust program being optimized, and may be
subjected to unpredictable side-effects from the outside world.

And in that case, it is useful to be able to assert precise manual control on the
memory accesses that are carried out by a Rust program in a certain memory
region. This is the purpose of _volatile memory operations_, which allow a Rust
programmers to generate a carefully controlled stream of hardware memory load
and store instructions, which is guaranteed to be left untouched by the Rust
compiler's optimizer even though surrounding Rust code will continue to be
optimized as usual.

---

Volatile memory operations are exposed in the `std::volatile` module of the Rust
standard library, or alternatively in the `core::volatile` module for the
purpose of writing `#![no_std]` applications. They are interfaced through
fixed-size data wrappers that are somewhat reminiscent of the API used for
atomic operations in `std::sync::atomic`:

```rust
use std::sync::atomic::Ordering;
use std::ptr::NonNull;
use std::volatile::VolatileU8;

unsafe fn do_volatile_things(target: NonNull<VolatileU8>) -> u8 {
    target.store(42, Ordering::Relaxed);
    target.load_not_atomic()
}
```

Notice that volatile types must be manipulated via pointers, instead of
Rust references. These unusual and unpleasant ergonomics are necessary in order
to achieve the desired semantics of manually controlling every access to the
target memory location, because the mere existence of a Rust reference pointing
to a memory region allows the Rust compiler to generate memory operations
targeting this region (be they prefetches, register spills, ...).

Because a Rust pointer is not subjected to borrow checking and has no obligation
of pointing towards a valid memory location, this means that using a Volatile
wrapper in any way is unsafe.

As a second difference, in addition to familiar atomic memory operations,
volatile types expose the `load_not_atomic()` and `store_not_atomic()` methods.
As their name suggest, these memory operations are not considered to be atomic
by the compiler, and are therefore not safe to concurrently invoke in multiple
threads.

On hardware with global cache coherence, a category which encompasses the vast
majority of Rust's supported compilation targets, use of these methods will
generate exactly the same code as using the `load()` and `store()` atomic access
methods with `Relaxed` memory ordering, with the only difference being that
data races are Undefined Behavior from the compiler's point of view. When that
is the case, safer `Relaxed` atomic volatile operations should be preferred to
their non-atomic equivalents.

Unfortunately, however, not all of Rust's compilation targets exhibit global
cache coherence. GPU hardware, such as the `nvptx` target, may only exhibit
cache coherence among local "blocks" of threads. And abstract machines like WASM
may not guarantee cache coherence at all without specific precautions. On those
compilation targets, `Relaxed` loads and stores may either be unavailable, or
lead to the generation of machine instructions more complex than native loads
and stores, which may not be wanted where maximal hardware control or CPU
performance is desired.

It is only for the sake of providing an alternative to the current
`ptr::[read|write]_volatile` mechanism on such platforms that the
`[load|store]_not_atomic()` functions are being proposed, and they should not
be used where better alternative exists.

Finally, unlike with atomics, the compiler is not allowed to optimize the above
function into the following...

```rust
unsafe fn do_volatile_things(target: NonNull<VolatileU8>) -> u8 {
    target.store(42, Ordering::Relaxed);
    42
}
```

...or even the following, which it could normally do if it the optimizer managed
to prove that no other thread has access to the `VolatileU8` variable:

```rust
unsafe fn do_volatile_things(_target: NonNull<VolatileU8>) -> u8 {
    42
}
```

This is the definining characteristic of volatile operations, which makes them
suitable for sensitive memory manipulations such as cryptographic secret erasure
or memory-mapped I/O.

---

Experienced Rust users may be familiar with the previously existing
`std::ptr::read_volatile()` and `std::ptr::write_volatile()` functions, or
equivalent methods of raw pointer objects, and wonder how the new facilities
provided by `std::volatile` compare to those methods.

The answer is that this new volatile data access API supersedes its predecessor,
which is now _deprecated_, by improving upon it in several ways:

- The data race semantics of `Relaxed` volatile data accesses more closely
  matches the data race semantics of most hardware, and therefore eliminates an
  unnecessary mismatch between volatile semantics and low-level hardware load
  and store semantics when it comes to concurrency.
- `VolatileXyz` wrappers are only supported for data types which are supported
  at the machine level. Therefore, one no longer needs to worry about the
  possibility of the compiler silently turning a Rust-level volatile data access
  into multiple hardware-level memory operations. In the same manner as with
  atomics, if a `Volatile` wrapper type is provided by Rust, the underlying
  hardware is guaranteed to support memory operations of that width.
- The ability to specify stronger-than-`Relaxed` memory orderings and to use 
  memory operations other than loads and stores enables Rust to draw a clear
  distinction between atomic operations which are meant to synchronize normal
  Rust code and atomic operations which are meant to synchronize with arbitrary
  FFI edge cases (such as threads spawned by LD_PRELOAD unbeknownst to the Rust
  compiler), which in turn would enable better optimization of atomic operations
  in the vast majority of Rust programs, as will be further discussed in the
  Unresolved Questions section.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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

Using LLVM's `Relaxed` atomic volatile operations instead resolves both problems
on globally cache-coherent hardware where native loads and stores have `Relaxed`
or stronger semantics. The vast majority of hardware which Rust supports today
and is expected to support in the future exhibits global cache coherence,
so making volatile feel more at home on such hardware is a sizeable achievement.

For exotic platforms whose basic memory loads and stores do not guarantee global
cache coherence, such as `nvptx`, this RFC adds `load_not_atomic()` and
`store_not_atomic()` operations. It is unclear at this point in time whether
these two methods should be stabilized, or an alternative solution such as
extending Rust's atomic operation model with synchronization guarantees weaker
than `Relaxed` should be researched.

As this feels like a complex and niche edge case that should not block the most
generally useful subset of volatile atomic operations, this RFC proposes to
implement these operations behind a different feature gate, and postpone their
stabilization until supplementary research has determined whether they are
truly a necessary evil or not.

---

Switching to LLVM's atomic volatile accesses without also changing the API of
`ptr::[read|write]_volatile` would unfortunately not resolve the memory access
tearing problem very well, because although oversized volatile accesses would
not compile anymore, this fact would only be "reported" to the user via an LLVM
crash. This is not a nice user experience, which is why volatile wrapper types
are proposed instead.

Their design largely mirrors that of Rust's existing atomic types, which is only
appropriate since they do expose atomic operations. The current proposal would
be to fill the newly built `std::volatile` module with the following entities
(some of which may not be available on a given platform, we will come back to
this point in the moment):

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

The API of these volatile types would then be very much like that of existing
`AtomicXyz` types, except for the fact that it would be based on raw pointers
instead of references because the existence of a Rust reference to a memory
location is fundamentally at odds with the precise control of hardware load and
store generation that is required by volatile use case.

To give a more concrete example, here is what the API of `VolatileBool` would
look like on a platform with full support for this type.

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

    // NOTE: Unlike with `AtomicBool`, `get_mut()` and `into_inner()` operations
    //       are not provided, because it is never safe to assume that no one
    //       is concurrently accessing volatile data. As an alternative, these
    //       operations could be provided in an unsafe way, if someone can find
    //       a use case for them.

    /// Load a value from the bool
    ///
    /// `load` takes an `Ordering` argument which describes the memory ordering
    /// of this operation. Possible values are SeqCst, Acquire and Relaxed.
    ///
    /// # Panics
    ///
    /// Panics if order is Release or AcqRel.
    ///
    /// # Safety
    ///
    /// The `self` pointer must be well-aligned and point to a valid memory
    /// location containing a valid `bool` value.
    ///
    pub unsafe fn load(self: NonNull<Self>, order: Ordering) -> bool { /* ... */ }

    // ... and then a similar transformation is carried out on all other atomic
    // operation APIs from `AtomicBool`:

    pub unsafe fn store(self: NonNull<Self>, val: bool, order: Ordering) { /* ... */ }

    pub unsafe fn swap(self: NonNull<Self>, val: bool, order: Ordering) -> bool { /* ... */ }

    pub unsafe fn compare_and_swap(
        self: NonNull<Self>,
        current: bool,
        new: bool,
        order: Ordering
    ) -> bool { /* ... */ }

    pub unsafe fn compare_exchange(
        self: NonNull<Self>,
        current: bool,
        new: bool,
        success: Ordering,
        failure: Ordering
    ) -> Result<bool, bool> { /* ... */ }

    pub unsafe fn compare_exchange_weak(
        self: NonNull<Self>,
        current: bool,
        new: bool,
        success: Ordering,
        failure: Ordering
    ) -> Result<bool, bool> { /* ... */ }

    pub unsafe fn fetch_and(self: NonNull<Self>, val: bool, order: Ordering) -> bool { /* ... */ }

    pub unsafe fn fetch_nand(self: NonNull<Self>, val: bool, order: Ordering) -> bool { /* ... */ }

    pub unsafe fn fetch_or(self: NonNull<Self>, val: bool, order: Ordering) -> bool { /* ... */ }

    pub unsafe fn fetch_xor(self: NonNull<Self>, val: bool, order: Ordering) -> bool { /* ... */ }

    // Finally, non-atomic load and store operations are provided:

    /// Load a value from the bool in a non-atomic way
    ///
    /// This method is provided for the sake of supporting platforms where
    /// `load(Relaxed)` either is unsupported or compiles down to more than a
    /// single hardware load instruction. As a counterpart, it is UB to use it
    /// in a concurrent setting. Use of `load(Relaxed)` should be preferred
    /// whenever possible.
    ///
    /// # Safety
    ///
    /// The `self` pointer must be well-aligned, and point to a valid memory
    /// location containing a valid `bool` value.
    ///
    /// Using this operation to access memory which is concurrently written by
    /// another thread is a data race, and therefore Undefined Behavior.
    ///
    pub unsafe fn load_not_atomic(self: NonNull<Self>) -> bool { /* ... */ }

    // ...and we have the same idea for stores:
    pub unsafe fn store_not_atomic(self: NonNull<Self>, val: bool) { /* ... */ }
}
```

---

Like `std::sync::atomic::AtomicXyz` APIs, the `std::volatile::VolatileXyz` APIs
are not guaranteed to be fully supported on every platform. Cases of partial
platform support which are shared with `AtomicXyz` APIs include:

- Not supporting atomic accesses to certain types like `u64`.
- Not supporting atomic read-modify-write operations like `swap()`.

In addition, a concern which is specific to `Volatile` types is the case where
atomic operations are not supported at all, but non-atomic volatile operations
are supported. In this case, the `AtomicBool` type above would only expose
the `load_not_atomic()` and `store_not_atomic()` methods.

Platform support for volatile operations can be queried in much of the
same way as platform support for atomic operations:

- `#[cfg(target_has_atomic = N)]`, which can be used today to test full support
  for atomic operations of a certain width N, may now also be used to test
  full support for volatile atomic operations of the same width.
- `#[cfg(target_has_atomic_load_store = N)]` may similarly be used to test
  support for volatile atomic load and store operations.
- A new cfg directive, `#[cfg(target_has_volatile = N)]`, may be used to test
  support for non-atomic loads and stores of a certain width (i.e. `_not_atomic`
  operations).

This latter directive can be initially pessimistically implemented as a synonym
of `#[cfg(target_has_atomic_load_store = N)]`, then gradually extended to
support targets which have no or non-native support of `Relaxed` atomics
but do have native load/store instructions of a certain width, such as `nvptx`.

In this way, the proposed volatile atomic operation API can largely re-use the
already existing atomic operation support infrastructure, which will greatly
reduce effort duplication between these two closely related functionalities.

---

This RFC currently proposes to expose the full power of LLVM's atomic volatile
operations, including e.g. read-modify-write operations like compare-and-swap,
because it is consistent with the atomics operation API and could have
legitimate uses in interprocess communication scenarios, as a marker of the
nuance between well-optimized program-local synchronization and worst-case FFI
synchronization. See Unresolved Questions section for more details.

However, the fact that these operations do not necessarily compile into a single
hardware instruction is arguably a footgun for volatile's use cases, and it
could be argued that initially only stabilizing loads, stores and `Relaxed`
atomic ordering would be more prudent. This will be revisited in the
alternatives section of this RFC.

---

As currently designed, this RFC uses `arbitrary_self_types` to give method-like
semantics to a `NonNull` raw pointer. This seems necessary to get reasonable
ergonomics with an atomics-like wrapper type approach. However, it could also be
argued that a `VolatileXyz::store(ptr, data, ordering)` style of API would work
well enough, and avoid coupling with unstable features. Similarly, the use of
`NonNull` itself could be debated. This will be revisited in the alternatives
section of the RFC as well.


# Drawbacks
[drawbacks]: #drawbacks

Deprecating the existing volatile operations will cause language churn. Not
deprecating them will keep around two different and subtly incompatible ways to
do the same thing, which is equally bad. It could be argued that the issues with
the existing volatile operations, while real, do not warrant the full complexity
of this RFC.

As mentioned above, exposing more than loads and stores, and non-`Relaxed`
atomic memory orderings, also muddies the "a volatile op should compile into one
hardware memory instruction" narrative that is so convenient for loads and
stores. Further, compatibility of non-load/store atomics in IPC scenario may
require some ABI agreement on how atomics should be implemented between the
interacting programs.

If this is perceived to be a problem, we could decide to do away with some of
the complexity by initially focusing on a restricted subset of this proposal
which only supports `Relaxed` loads and stores, and saving more complex atomic
operations as a future extension.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This effort seems worthwhile because it simultaneously eliminates two well-known
and unnecessary volatile footguns (data races and tearing) and opens interesting
new possibilities in interprocess communication.

But although this feature may seem simple, its design space is actually
remarquably large, and a great many alternatives were considered before reaching
the current design proposal. Here are some design knobs that were explored:

## Extending `AtomicXyz`
[extending-atomicxyz]: #extending-atomicxyz

Atomic volatile operations could be made part of the Atomic wrapper types.
However, they cannot be just another atomic memory ordering, due to the fact
that atomic ops take an `&self` (which is against the spirit of volatile as it
is tagged with `dereferenceable` at the LLVM layer).

Instead, one would need to add more methods to atomic wrapper types, which take
a pointer as a self-parameter. This API inconsistency was felt to be
unnecessarily jarring to users, not to mention that potentially having a copy of
each atomic memory operation with a `_volatile` prefix would reduce the
readability of the `AtomicXyz` types' API documentation.

In addition, it is extremely common to want either all operations on a memory
location to be volatile, or none of them. Providing separate wrapper types
helps enforce this very common usage pattern at the API level.

## Self-type or not self-type
[self-type]: #self-type

Instead of using `arbitrary_self_types` to get `a.load(Ordering::Relaxed)`
method syntax on pointer-like objects, one could instead provide the volatile
operations as inherent methods, i.e. `VolatileU8::load(a, Ordering::Relaxed)`.

This has the advantage of avoiding coupling this feature to another unstable
feature. But it has the drawback of being incredibly verbose. Typing the same
volatile type name over and over again in a complex transaction would certainly
get old and turn annoying quickly, and we don't want anger to distract low-level
developers from the delicate task of implementing the kind of subtle algorithm
that requires volatile operations.

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

## Full atomics vocabulary vs sticking with hardware semantics
[atomics-vs-hardware]: #atomics-vs-hardware

Currently, this RFC basically proposes exposing a volatile version of every
atomic operation supported by Rust for maximal expressive power.

But it could also be argued that this distracts us from volatile's main purpose
of generating a stream of simple hardware instructions without using inline
assembly:

- Non-`Relaxed` atomics will entail memory barriers
- Compare-and-swap may be implemented as a load-linked/store-conditional loop
- Some types like `VolatileBool` are dangerous when interacting with untrusted
  code because they come with data validity invariants.

From this perspective, there would be an argument in favor of only supporting
`Relaxed` load/stores and machine data types, at least initially. In this case,
one could split this feature into three feature gates:

- `Relaxed` volatile atomic loads and stores, which are most urgently needed.
- Non-`Relaxed` orderings and read-modify-write atomics, which open new
  possibilities for shared-memory IPC.
- `_not_atomic()` operations, where it is not yet clear whether the proposed API
  is even the right solution to the problem being solved, and more research is
  needed before reaching a definite conclusion.


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

## Untrusted shared-memory IPC

Although it performs a step in the right direction by strengthening the
definition of volatile accesses to result the amount of possible avenues for
undefined behavior, this RFC will no fully resolve the "untrusted shared memory"
use case, where Rust code is interacting with untrusted arbitrary code via a
shared memory region.

Doing so would also require work on clarifying LLVM semantics so that it is
absolutely clear that a malicious process cannot cause UB in another process by
by writing data in memory that's shared between the two, no matter if said
writes are non-atomic, non-volatile, etc.

## Necessity of non-atomic operations

The necessity of having `load_not_atomic()` and `store_not_atomic()` methods,
as opposed to alternatives such as weaker-than-`Relaxed` atomics, should be
researched before stabilizing that subset of this RFC.


# Future possibilities
[future-possibilities]: #future-possibilities

As mentioned above, this RFC is a step forward in addressing the untrusted
shared-memory IPC use case that is of interest to many "supervisor" programs,
but not the end of that story. Finishing it will likely require LLVM assistance.

If we decide to drop advanced atomic orderings and operations from this RFC,
then they will fall out in this section.

This RFC would also benefit from a safer way to interact with volatile memory
regions than raw pointers.

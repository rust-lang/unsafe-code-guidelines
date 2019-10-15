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
(mostly) unoptimized generation of a matching stream of hardware load and store
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
   store instructions.

By implementing support for LLVM's atomic volatile operations, and encouraging
their use on every hardware that supports them, we eliminate these divergences
from hardware behavior and therefore bring volatile operations closer to the
"defer to hardware memory model" semantics that programmers expect them to have.
This reduces the odd of mistake in programs operating outside of the regular
Rust Abstract Machine semantics, which are notoriously hard to get right.

As an attractive side-effect, atomic volatile memory operations also enable
higher-performance interprocess communication between mutually trusting Rust
programs through lock-free synchronization of shared memory objects.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Rust compiler generally assumes that the program that it is building is
living in a fully isolated memory space. It leverages this knowledge to
transform said program's memory access patterns for performance optimization
purposes, under the assumption that said transformations will not have any
externally observable effect other than the program running faster.

Examples of such transformations include:

- Caching data from RAM into CPU registers.
- Eliminating unused loads and unobservable stores.
- Merging neighboring stores with each other.
- Only updating the part of an over-written struct that has actually changed.

Although these optimizations are most of the time correct and useful, there are
some situations where they are inappropriate, including but not limited to:

- [Memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O), a common
  low-level communication protocol between CPUs and peripherals, where hardware
  registers masquerading as memory can be used to program said hardware by
  accessing the registers in very specific load/store patterns.
- [Shared memory](https://en.wikipedia.org/wiki/Shared_memory), a form of
  inter-process communication where two programs can communicate via a common
  memory block, and it is therefore not appropriate for Rust to assume that it
  is aware of all memory accesses occurring inside said memory block.
- [Cryptography](https://en.wikipedia.org/wiki/Cryptography), where it is
  extremely important to ensure that sensitive information is erased after use,
  and is not leaked via indirect means such as recognizable scaling patterns in
  the time taken by a system to process attacker-crafted inputs.

In such circumstances, it may be necessary to assert precise manual control on
the memory accesses that are carried out by a Rust program in a certain memory
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

Several specificities, however, should be apparent from the above usage example.

First of all, volatile types must be manipulated via pointers, instad of Rust
references. These unusual and unpleasant ergonomics are necessary in order to
achieve the desired semantics of manually controlling every access to the target
memory location, because the mere existence of a Rust reference pointing to a
memory region allows the Rust compiler to generate memory operations targeting
this region (be they prefetches, register spills...).

Because a Rust pointer is not subjected to borrow checking and has no obligation
of pointing towards a valid memory location, this means that using a Volatile
wrapper in any way is unsafe.

Second, in addition to familiar atomic memory operations, volatile types expose
the `load_not_atomic()` and `store_not_atomic()` methods. As their name suggest,
these memory operations are not considered to be atomic by the compiler, and
are therefore not safe to concurrently invoke in multiple threads.

On the vast majority of hardware platforms supported by Rust, using these
methods will generate exactly the same code as using the `load()` and `store()`
methods with `Relaxed` memory ordering, with the only difference being that
data races are Undefined Behavior. When that is the case, safer `Relaxed` atomic
volatile operations should be preferred to their non-atomic equivalents.

But unfortunately, Rust supports a couple of platforms, such as the `nvptx`
assembly used by NVidia GPUs, where `Relaxed` atomic ordering is either
unsupported or emulated via a stream of hardware instructions that is more
complex than plain loads and stores. Supporting not-atomic volatile loads and
stores is necessary to get minimal `Volatile` support on those platforms.

Finally, unlike with atomics, the compiler is not allowed to optimize the above
function into the following...

```rust
unsafe fn do_volatile_things(target: NonNull<VolatileU8>) -> u8 {
    target.store(42, Ordering::Relaxed);
    42
}
```

...or even the following, if it can prove that no other thread has access to
the underlying `VolatileU8` variable:

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
which is now _deprecated_, by improving upon it in many different ways:

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
  memory operations other than loads and stores enables new use cases which were
  not achievable before without exploiting Undefined Behavior, such as
  high-performance synchronization of mutually trusting Rust processes
  via lock-free data structures in shared memory.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The fundamental purpose of volatile operations, in a system programming language
like Rust, is to allow a developer to locally escape the Rust Abstract Machine's
weak memory semantics and defer to the hardware's memory semantics instead,
without resorting to the full complexity, overhead, and non-portability of
assembly (inline or otherwise).

Rust's current volatile operations, which defer to LLVM's non-atomic volatile
operations, do not achieve this goal very well because...

1. Their data race semantics do not match the data race semantics of the
   hardware which volatile is supposed to defer to, and as a result are
   unnecessarily surprising and unsafe. They prevent some synchronization
   patterns which are legal at the hardware level but not at the abstract
   machine level, such as the "seqlock", to be expressed using volatile
   operations. No useful optimization opportunities are opened by this undefined
   behavior, since volatile operations cannot be meaningfully optimized.
2. The absence of a hard guarantee that each volatile load or store will
   translate into exactly one load or store at the hardware level is needlessly
   tracherous in scenarios where memory access patterns must be very precisely
   controlled, such as memory-mapped I/O.

Using LLVM's `Relaxed` atomic volatile operations instead resolves both problems
on cache-coherent hardware where native loads and stores have `Relaxed` or
stronger semantics. The vast majority of hardware which Rust supports today and
is expected to support in the future is cache-coherent, and therefore making
volatile feel more at home on such hardware is a major win.

However, we obviously must still have something for those exotic platforms whose
basic memory loads and stores are not cache-coherent, such as `nvptx`. Hence the
compromise of `load_not_atomic()` and `store_not_atomic()` is still kept around,
only discouraging its use.

---

It should be noted that switching to LLVM's atomic volatile accesses does not
resolve the second problem very well per se, because although oversized volatile
accesses will not compile anymore, they will only be "reported" to the user via
LLVM crashes. This is not a nice user experience, which is why volatile wrapper
types are proposed.

Their design largely mirrors that of Rust's existing atomic types, which is only
appropriate since they do expose atomic operations. One goal of this design is
that it should be possible to re-use the architecture-specific code that already
exists to selectively expose Rust atomic types and operations depending on what
the hardware supports under the hood.

---

This RFC currently proposes to expose the full power of LLVM's atomic volatile
operations, including e.g. read-modify-write operations like compare-and-swap,
because there are legitimately useful use cases for these operations in
interprocess communication scenarios.

However, the fact that these operations do not necessarily compile into a single
hardware instruction is arguably a footgun for volatile's use cases, and it
could be argued that initially only stabilizing loads, stores and `Relaxed`
atomic ordering would be more prudent. I'll go back to this in the alternatives
section of this RFC.

---

As currently designed, this RFC uses `arbitrary_safe_types` to give method-like
semantics to a `NonNull` raw pointer. I believe that this is necessary to get
reasonable ergonomics with an atomics-like wrapper type approach. However, it
could also be argued that a `VolatileXyz::store(ptr, data, ordering)` style of
API could reasonably work, and avoid coupling with unstable features. Similarly,
the use of `NonNull` itself could be debated. I'll come back to these points in
the alternatives section as well.


# Drawbacks
[drawbacks]: #drawbacks

Deprecating the existing volatile operations will cause language churn. Not
deprecating them will keep around two different and subtly incompatible ways to
do the same thing, which is equally bad. It could be argued that the issues with
the existing volatile operations, while real, do not warrant the full complexity
of this RFC.

Atomic volatile operations are also a somewhat LLVM-specific concept, and
requiring them may make the life of our other compiler backends harder.

As mentioned above, exposing more than loads and stores, and non-`Relaxed`
atomic memory orderings, also muddies the "a volatile op should compile into one
hardware memory instruction" that is so convenient for loads and stores.
Further, compatibility of non-load/store atomics in IPC scenario may require
some ABI agreement between the interacting programs.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

I think we want to do this because it simultaneously eliminates two well-known
and unnecessary volatile footguns (data races and tearing) and opens interesting
new possibilities.

But although this feature may seem simple, its design space is actually
remarquably large, and a great many alternatives were considered before reaching
the current design proposal. Here is a map of some design knobs that I explored:

## Extending `AtomicXyz`
[extending-atomicxyz]: #extending-atomicxyz

Atomic volatile operations could be made part of the Atomic wrapper types.
However, they cannot be just another atomic memory ordering, due to the fact
that atomic ops take an `&self` (which is against the spirit of volatile as it
is tagged with `dereferenceable` at the LLVM layer).

Instead, one would need to add more methods to atomic wrapper types, which take
a pointer as a self-parameter. I thought that this API inconsistency would be
jarring to users, not to mention that potentially having a copy of each atomic
memory operation with a `_volatile` prefix would get annoying quickly when
reading through the API docs.

## Self-type or not self-type
[self-type]: #self-type

Instead of using `arbitrary_self_types` to get `a.load(Ordering::Relaxed)`
method syntax on pointer-like objects, one could instead provide the volatile
operations as inherent methods, i.e. `VolatileU8::load(a, Ordering::Relaxed)`.

This has the advantage of avoiding coupling this feature to another unstable
feature. But it has the drawback of being incredibly verbose. Typing the same
volatile type name over and over again in a complex shared memory transaction
would certainly get old and turn annoying quickly, and we don't want anger to
distract low-level developers from the delicate task of implementing the kind
of subtle algorithm that requires volatile operations.

## `NonNull<T>` vs `*mut T` vs `*const T` vs other
[pointers]: #pointers

Honestly, I don't have a very strong opinion there. I have a positive _a priori_
towards `NonNull<T>` because it encodes an important invariant in the API, and
I think that arbitrary self types make its usually poor ergonomics bearable.

But what I would really want here is something like a non-`dereferenceable`
Rust reference. I don't like the fact that I have to give up on the borrow
checker and make everything unsafe just to do volatile loads and stores, which
are not unsafe _per se_ as long as they occur via a `Volatile` wrapper. It just
feels like we should be able to propose something better than full references
vs full raw pointers here...

## Full atomics vocabulary vs hardware semantics
[atomics-vs-hardware]: #atomics-vs-hardware

Currently, this RFC basically proposes exposing a volatile version of every
atomic operation supported by Rust for maximal expressive power, and I could
definitely find uses for the new possibilities that this opens in IPC scenarios.

But it could also be argued that this distracts us from volatile's main purpose
of generating a stream of simple hardware instructions without using inline
assembly:

- Non-`Relaxed` atomics will entail memory barriers
- Compare-and-swap may be implemented as a load-linked/store-conditional loop
- Some types like `VolatileBool` are dangerous when interacting with untrusted
  code because they come with data validity invariants.

From this perspective, there would be an argument in favor of only supporting
`Relaxed` load/stores and machine data types, at least initially. And I could
get behind that. But since the rest can be useful in IPC with trusted Rust code,
I thought it might be worth at least considering.


# Prior art
[prior-art]: #prior-art

As far as I know, LLVM's notion of atomic volatile, which is being exposed here,
is rather original. The closest thing that it makes me think about is how Java
uses the `volatile` keyword for what most other languages call atomics. But
I'm not sure if Java also retains the other semantics of `volatile` in the C
family (e.g. "cannot be optimized out ever").

There is a lot more prior art behind C's notion of volatile, which is closer to
Rust's current notion of volatile. That being said, most discussions of C/++
volatile semantics end up in complaints about how ill-specified, how easy it is
to get it wrong, how "contagious" it can get... so I'm not sure if it's
something we want to emulate. Besides, Rust's notion of volatile differs
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
believed to be unsound due as they expose an `&self` to the sensitive memory
region, but `voladdress` is believed not to be affected by this problem.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

I expect the RFC process to be an occasion to revisit some points of the
rationale and alternative sections better than I can do on my own, bring new
arguments and edge cases on the table, and more generally refine the API.

If we decide to implement this, implementation should be reasonably
straightforward and uneventful, as most of the groundwork has already been done
over the course of implementing `ptr::read_volatile()`, `ptr::write_volatile()`
and `std::sync::atomic` (as far as I know at least).

This RFC will no fully resolve the "untrusted shared memory" use case, because
doing so also requires work on clarifying LLVM semantics so that it is
absolutely clear that a malicious process cannot cause UB in another process
by writing data in memory that's shared between the two, no matter if said
writes are non-atomic, non-volatile, etc.


# Future possibilities
[future-possibilities]: #future-possibilities

If we decide to drop advanced atomic orderings and operations from this RFC,
then they will fall out in this section.

This RFC would also benefit from a safer way to interact with volatile memory
regions than raw pointers.

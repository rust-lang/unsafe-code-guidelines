# Rust Memory Interface

**Note:** This document is not normative nor endorsed by the UCG WG. Its purpose is to be the basis for discussion and to set down some key terminology.

The purpose of this document is to describe the interface between a Rust program and memory.
This interface is a key part of the Rust Abstract Machine: it lets us separate concerns by splitting the Machine (i.e., its specification) into two pieces, connected by this well-defined interface:
* The *expression/statement semantics* of Rust boils down to explaining which "memroy events" (calls to the memory interface) happen in which order - expressed as calls to the methods of this interface, and reactions to its return values. This part of the specification is *pure* in the sense that it has no "state": everything that needs to be remembered from one expression evaluation to the next is communicated through memory.
* The Rust *memory model* explains which interactions with the memory are legal (the others are UB), and which values can be returned by reads. A memory model is defined by implementing the memory interface.

The interface shown below is also opinionated in several ways.
It is not intended to be able to support *any imaginable* memory model, but rather start the process of reducing the design space of what we consider a "reasonable" memory model for Rust.
For example, it explicitly acknowledges that pointers are not just integers and that uninitialized memory is special (both are true for C and C++ as well but you have to read the standard very careful, and consult [defect report responses](http://www.open-std.org/jtc1/sc22/wg14/www/docs/dr_260.htm), to see this).
Another key property of the interface presented below is that it is *untyped*.
This implies that in Rust, *operations are typed, but memory is not* - a key difference to C and C++ with their type-based strict aliasing rules.
At the same time, the memory model provides a *side-effect free* way to turn pointers into "raw bytes", which is *not* [the direction C++ is moving towards](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2364.pdf), and we might have to revisit this choice later if it turns out to not be workable.

## Pointers

One key question a memory model has to answer is *what is a pointer*.
It might seem like the answer is just "an integer of appropriate size", but [that is not the case][pointers-complicated].
This becomes even more prominent with aliasing models such as [Stacked Borrows].
So we leave it up to the memory model to answer this question, and make `Pointer` an associated type.
Practically speaking, `Pointer` will be some representation of an "address", plus [provenance] information.

[provenance]: https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/glossary.md#pointer-provenance

## Bytes

The unit of communication between the memory model and the rest of the program is a *byte*.
Again, the question of "what is a byte" is not as trivial as it might seem; beyond `u8` values we have to represent `Pointer`s and [uninitialized memory][uninit].
We define the `Byte` type as follows, where `Pointer` will later be instantiated with the `Memory::Pointer` associated type.

```rust
enum Byte<Pointer> {
    /// The "normal" case: a (frozen, initialized) integer in `0..256`.
    Raw(u8),
    /// An uninitialized byte.
    Uninit,
    /// One byte of a pointer.
    PtrFragment {
        /// The pointer of which this is a byte.
        /// That is, the byte is a fragment of this pointer.
        ptr: Pointer,
        /// Which byte of the pointer this is.
        /// `idx` will always be in `0..PTR_SIZE`.
        idx: u8,
    }
}
```

The purpose of `PtrFragment` is to enable a byte-wise representation of a `Pointer`.
On a 32-bit system, the sequence of 4 bytes representing `ptr: Pointer` is:
```
[
    PtrFragment { ptr, idx: 0 },
    PtrFragment { ptr, idx: 1 },
    PtrFragment { ptr, idx: 2 },
    PtrFragment { ptr, idx: 3 },
]
```

Based on the `PtrToInt` trait (see below), we can turn every initialized `Byte` into an integer in `0..256`:

```rust
impl<Pointer: PtrToInt> Byte<Pointer> {
    fn as_int(self) -> Option<u8> {
        match self {
            Byte::Raw(int) => Some(int),
            Byte::Uninit => None,
            Byte::PtrFragment { ptr, idx } =>
                Some(ptr.get_byte(idx)),
        }
    }
}
```

## Memory interface

The Rust memory interface is described by the following (not-yet-complete) trait definition:

```rust
/// All operations are fallible, so they return `Result`.  If they fail, that
/// means the program caused UB. What exactly the `UndefinedBehavior` type is
/// does not matter here.
type Result<T=()> = std::result::Result<T, UndefinedBehavior>;

/// *Note*: All memory operations can be non-deterministic, which means that
/// executing the same operation on the same memory can have different results.
/// We also let all operations potentially mutate memory. For example, reads
/// actually do change the current state when considering concurrency or
/// Stacked Borrows.
/// This is pseudo-Rust, so we just use fully owned types everywhere for
/// symmetry and simplicity.
trait Memory {
    /// The type of pointer values.
    type Pointer: Copy + PtrToInt;

    /// The size of pointer values.
    const PTR_SIZE: u64;

    /// Create a new allocation.
    fn allocate(&mut self, size: u64, align: u64) -> Result<Self::Pointer>;

    /// Remove an allocation.
    fn deallocate(&mut self, ptr: Self::Pointer, size: u64, align: u64) -> Result;

    /// Write some bytes to memory.
    fn write(&mut self, ptr: Self::Pointer, bytes: Vec<Byte<Self::Pointer>>) -> Result;

    /// Read some bytes from memory.
    fn read(&mut self, ptr: Self::Pointer, len: u64) -> Result<Vec<Byte<Self::Pointer>>>;

    /// Offset the given pointer.
    fn offset(&mut self, ptr: Self::Pointer, offset: u64, mode: OffsetMode)
        -> Result<Self::Pointer>;

    /// Cast the given integer to a pointer.  (The other direction is handled by `PtrToInt` below.)
    fn int_to_ptr(&mut self, int: u64) -> Result<Self::Pointer>;
}

/// The `Pointer` type must know how to extract its bytes, *without any access to the `Memory`*.
trait PtrToInt {
    /// Get the `idx`-th byte of the pointer.  `idx` must be in `0..PTR_SIZE`.
    fn get_byte(self, idx: u8) -> u8;
}

/// The rules applying to this pointer offset operation.
enum OffsetMode {
    /// Wrapping offset; never UB.
    Wrapping,
    /// Non-wrapping offset; UB if it wraps.
    NonWrapping,
    /// In-bounds offset; UB if it wraps or if old and new pointer are not both
    /// in-bounds of the same allocation (details are specified by the memory
    /// model).
    Inbounds,
}
```

We will generally assume we have a particular memory model in scope, and freely refer to its `PTR_SIZE` and `Pointer` items.
We will also write `Byte` for `Byte<Pointer>`.

This is a very basic memory interface that is incomplete in at least the following ways:

* To implement rules like "dereferencing a null, unaligned, or dangling raw pointer is UB" (even if no memory access happens), there needs to be a way to do an "alignment, bounds and null-check".
* There needs to be some way to do alignment checks -- either using the above operation, or by adding `align` parameters to `read` and `write`.
* To represent concurrency, many operations need to take a "thread ID" and `read` and `write` need to take an [`Ordering`].
* To represent [Stacked Borrows], there needs to be a "retag" operation, and that one will in fact be "lightly typed" (it cares about `UnsafeCell`).
* Maybe we want operations that can compare pointers without casting them to integers.

[pointers-complicated]: https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html
[uninit]: https://www.ralfj.de/blog/2019/07/14/uninit.html
[`Ordering`]: https://doc.rust-lang.org/nightly/core/sync/atomic/enum.Ordering.html
[Stacked Borrows]: stacked-borrows.md

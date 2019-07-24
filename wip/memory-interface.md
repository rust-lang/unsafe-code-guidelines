# Rust Memory Interface

**Note:** This document is not normative nor endorsed by the UCG WG. Its purpose is to be the basis for discussion and to set down some key terminology.

The purpose of this document is to describe the interface between a Rust program and memory.
This interface is a key part of the Rust Abstract Machine: it lets us separate concerns by splitting the Machine (i.e., its specification) into two pieces, connected by this well-defined interface:
* The *expression/statement semantics* of Rust boils down to explaining which "memroy events" (calls to the memory interface) happen in which order.
* The Rust *memory model* explains which interactions with the memory are legal (the others are UB), and which values can be returned by reads.

The interface is also opinionated in several ways; this is not intended to be able to support *any imaginable* memory model, but rather start the process of reducing the design space of what we consider a "reasonable" memory model for Rust.
For example, it explicitly acknowledges that pointers are not just integers and that uninitialized memory is special (both are true for C and C++ as well but you have to read the standard very careful, and consult non-normative defect report responses, to see this).
Another key property of the interface presented below is that it is *untyped*.
This encodes the fact that in Rust, *operations are typed, but memory is not*---a key difference to C and C++ with their type-based strict aliasing rules.

## Pointers

One key question a memory model has to answer is *what is a pointer*.
It might seem like the answer is just "an integer of appropriate size", but [that is not the case][pointers-complicated].
This becomes even more prominent with aliasing models such as [Stacked Borrows].
So we will leave this question open, and treat `Pointer` as an "associated type" of the memory interface

## Bytes

The unit of communication between the memory model and the rest of the program is a *byte*.
Again the question of "what is a byte" is not as trivial as it might seem; beyond `u8` values we have to represent `Pointer`s and [uninitialized memory][uninit].
We define the `Byte` type (in terms of an arbitrary `Pointer` type) as follows:

```rust
enum Byte<Pointer> {
    /// The "normal" case: a (frozen, initialized) integer in `0..256`.
    Raw(u8),
    /// An uninitialized byte.
    Uninit,
    /// One byte of a pointer.
    PtrFragment {
        /// The pointer of which this is a byte.
        ptr: Pointer,
        /// Which byte of the pointer this is.
        /// `idx` will always be in `0..size_of::<usize>()`.
        idx: u8,
    }
}
```

The purpose of `PtrFragment` is to be able to have a byte-wise representation of a `Pointer`.
On a 32-bit system, the sequence of 4 bytes representing `ptr: Pointer` is:
```
[PtrFragment { ptr, idx: 0 }, PtrFragment { ptr, idx: 1 }, PtrFragment { ptr, idx: 2 }, PtrFragment { ptr, idx: 3 }]
```

## Memory interface

The Rust memory interface is described by the following (not-yet-complete) trait definition:

```rust
/// *Note*: All memory operations can be non-deterministic, which means that
/// executing the same operation on the same memory can have different results.
/// We also let all operations potentially mutated memory. For example, reads
/// actually do change the current state when considering concurrency or
/// Stacked Borrows.
/// And finally, all operations are fallible (they return `Result`); if they
/// fail, that means the program caused UB.
trait Memory {
    /// The type of pointer values.
    type Pointer;

    /// Create a new allocation.
    fn allocate(&mut self, size: u64, align: u64) -> Result<Self::Pointer, Error>;

    /// Remove an allocation.
    fn deallocate(&mut self, ptr: Self::Pointer, size: u64, align: u64) -> Result<(), Error>;

    /// Write some bytes to memory.
    fn write(&mut self, ptr: Self::Pointer, bytes: Vec<Byte<Self::Pointer>>) -> Result<(), Error>;

    /// Read some bytes from memory.
    fn read(&mut self, ptr: Self::Pointer, len: u64) -> Result<Vec<Byte<Self::Pointer>>, Error>;

    /// Offset the given pointer.
    fn offset(&mut self, ptr: Self::Pointer, offset: u64, mode: OffsetMode) -> Result<Self::Pointer, Error>;

    /// Cast the given pointer to an integer.
    fn ptr_to_int(&mut self, ptr: Self::Pointer) -> Result<u64, Error>;

    /// Cast the given integer to a pointer.
    fn int_to_ptr(&mut self, int: u64) -> Result<Self::Pointer, Error>;
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

This is a very basic memory interface that is incomplete in at least the following ways:

* To implement rules like "dereferencing a null, unaligned, or dangling raw pointer is UB" (even if no memory access happens), there needs to be a way to do an "alignment, bounds and null-check".
* There needs to be some way to do alignment checks -- either using the above operation, or by adding `align` parameters to `read` and `write`.
* To represent concurrency, many operations need to take a "thread ID" and `read` and `write` need to take an [`Ordering`].
* To represent [Stacked Borrows], there needs to be a "retag" operation, and that one will in fact be "lightly typed" (it cares about `UnsafeCell`).
* Maybe we want operations that can compare pointers without casting them to integers.

But I think it can still be useful to provide some basic terminology and grounds for further discussion.

[pointers-complicated]: https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html
[uninit]: https://www.ralfj.de/blog/2019/07/14/uninit.html
[`Ordering`]: https://doc.rust-lang.org/nightly/core/sync/atomic/enum.Ordering.html
[Stacked Borrows]: stacked-borrows.md

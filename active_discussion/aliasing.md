# Aliasing and memory model

## Brief Summary

Rust's borrow checker enforces some particular aliasing
requirements. For example, an `&u32` reference is always guaranteed to
be valid (dereferenceable) and immutable while it is in active
use. Similarly, an `&mut` reference cannot alias any other active
reference. It is less clear how those invariants translate to unsafe
code that is using raw pointers.

## See also

- ACA model https://github.com/nikomatsakis/rust-memory-model/issues/26
- Capability-based model https://github.com/nikomatsakis/rust-memory-model/issues/28
- A formal C memory model supporting integer-pointer casts https://github.com/nikomatsakis/rust-memory-model/issues/30
- Promising semantics for relaxed-memory concurrency https://github.com/nikomatsakis/rust-memory-model/issues/32
- Tootsie Pop model https://github.com/nikomatsakis/rust-memory-model/issues/21

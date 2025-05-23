Some of the things we want people to do with Rust can currently not be expressed in LLVM in a way that is fully backed by the LLVM LangRef.
Let's collect a list of those cases here.

## List of LLVM assumptions not backed by the spec

- To implement `ptr.addr()`, we assume that a pointer-to-int transmute yields the address.
  The LangRef is quiet about this (as it is about almost everything related to provenance).
  Alive says that this yields poison.
- To implement the desired semantics for `MaybeUninit<$int>` we need a type of arbitrary size that can hold arbitrary data -- including provenance.
  LLVM currently has no such type, the only type that is fully guaranteed to support provenance is `ptr` and that has a fixed size.
- This one is not about current Rust but about possible future extensions:
  when LLVM returns `poison` for some operation, we can *not* say that this corresponds to `uninit` in Rust. We *must* declare this immediate UB.
  The reason for this is that LLVM does not really support `poison` being stored in memory; Rust's `uninit` can therefore only correspond to LLVM's `undef`.

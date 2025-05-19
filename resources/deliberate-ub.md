When a crate explicitly acknowledges that what it does is UB, but prefers keeping that code over UB-free alternatives (or there are no UB-free alternatives), that is always a concerning sign.
We should evaluate whether there truly is some use-case here that is not currently achievable in well-defined Rust, and work with crate authors on providing a UB-free solution.

## Known cases of deliberate UB

### Cases related to concurrency

* crossbeam's `AtomicCell` "fallback path" uses a SeqLock, which [is well-known to not be compatible with the C++ memory model](https://www.hpl.hp.com/techreports/2012/HPL-2012-68.pdf).
  Specifically the problem is [this non-atomic volatile read](https://github.com/crossbeam-rs/crossbeam/blob/5d07fe43540d7f21517a51813acd9332744e90cb/crossbeam-utils/src/atomic/atomic_cell.rs#L980) which can cause data races and hence UB.
  This would be fine if we either (a) adopted LLVM's handling of memory races (then the problematic read would merely return `undef` instead of UB due to a data race), or (b) added [bytewise atomic memcpy](https://github.com/rust-lang/rfcs/pull/3301) and used that instead of the non-atomic volatile load.
  This is currently *not* UB in the LLVM IR we generate, due to LLVM's different handling of read-write races and because the code carefully uses `MaybeUninit`.
* crossbeam's `AtomicCell` "fast path" uses the standard library `Atomic*` types to do atomic reads and writes of [*any type* that has the right size](https://github.com/crossbeam-rs/crossbeam/blob/master/crossbeam-utils/src/atomic/atomic_cell.rs#L928-L932).
  However, doing an `AtomicU32` read (returning a `u32`) on a `(u16, u8)` is unsound because the padding byte can be uninitialized.
  (Also, pointer provenance is lost, so `AtomicCell<*const T>` does not behave the way people expect.)
  To fix this we need to be able to perform atomic loads at type `MaybeUninit<u32>`.
  The LLVM IR we generate here contains `noundef` so this UB is not just a specification artifact.<br>
  Furthermore, `compare_exchange` will compare padding bytes, which leads to UB.
  It is not clear how to best specify a useful `compare_exchange` that can work on padding bytes,
  see the [discussion here](https://github.com/rust-lang/unsafe-code-guidelines/issues/449).<br>
  The alternative is to not use the "fast path" for problematic types (and fall back to the SeqLock), but that requires some way to query at `const`-time whether the type contains padding (or provenance).
  (Or of course one can use inline assembly, but it would be better if that was not required.)
* crossbeam's deque uses [volatile accesses that really should be atomic instead](https://github.com/crossbeam-rs/crossbeam/blob/5a154def002304814d50f3c7658bd30eb46b2fad/crossbeam-deque/src/deque.rs#L70-L88).
  They cannot use atomic accesses as those are not possible for arbitrary `T`.
  This would be resolved by bytewise atomic memcpy.

### Cases related to aliasing

* `yoke` and similar crates relying in "stable deref" properties cause various forms of aliasing trouble (such as [having `Box` that alias with things](https://github.com/unicode-org/icu4x/issues/2095), or [having references in function arguments that get deallocated while the function runs](https://github.com/unicode-org/icu4x/issues/3696)).
  This also violates the LLVM assumptions that come with `noalias` and `dereferenceable`.
  This could be fixed by [`MaybeDangling`](https://github.com/rust-lang/rfcs/pull/3336).
* The entire `async fn` ecosystem and every hand-implemented self-referential generator or future is unsound since the self-reference aliases the `&mut` reference to the full generator/future.
  This is currently hackfixed by making `Unpin` meaningful for UB; a proper solution would be to add something like [`UnsafeAliased`](https://github.com/rust-lang/rfcs/pull/3467).
* Stacked Borrows forbids a bunch of things that might be considered too restrictive (and that go well beyond LLVM `noalias`):
  strict subobject provenance [rules out the `&Header` pattern](https://github.com/rust-lang/unsafe-code-guidelines/issues/256) and also affects [raw pointers derived from references](https://github.com/rust-lang/unsafe-code-guidelines/issues/134);
  eager assertion of uniquess makes [even read-only functions such as `as_mut_ptr` dangerous when they take `&mut`](https://github.com/rust-lang/unsafe-code-guidelines/issues/133);
  `&UnsafeCell` surprisingly [requires read-write memory even when it is never written](https://github.com/rust-lang/unsafe-code-guidelines/issues/303).
  There is a bunch of code out there that violates these rules one way or another.
  All of these are resolved by [Tree Borrows](https://perso.crans.org/vanille/treebor/), though [some subtleties around `as_mut_ptr` do remain](https://github.com/rust-lang/unsafe-code-guidelines/issues/450).

### Other cases

* `gxhash` wants to do a vector-sized load that may go out-of-bounds, and didn't find a better solution than causing UB with an OOB load and then masking off the extra bytes.
  See [here](https://github.com/ogxd/gxhash/issues/82) for some discussion and [here](https://github.com/ogxd/gxhash/blob/9eb19b021ff94a7b37beb5f479880d07e029b933/src/gxhash/platform/mod.rs#L18) for the relevant code.
  The same [also happens in `compiler-builtins`](https://github.com/rust-lang/compiler-builtins/issues/559).

## Former cases of deliberate UB that have at least a work-in-progress solution to them

* Various `offset_of` implementations caused UB by using `mem::uninitialized()`, or they used `&(*base).field` or `addr_of!((*base).field)` to project a dummy pointer to the field which is UB due to out-of-bounds pointer arithmetic.
  The `memoffset` crate has a sound implementation that however causes stack allocations which the compiler must optimize away.
  This will be fixed properly by the native [`offset_of!` macro](https://github.com/rust-lang/rfcs/pull/3308), which is [currently in nightly](https://github.com/rust-lang/rust/issues/106655).
* It used to be common to unwind out of `extern "C"` functions which is UB, see [this discussions](https://internals.rust-lang.org/t/unwinding-through-ffi-after-rust-1-33/9521).
  This is fixed by `extern "C-unwind"`, which is stable since Rust 1.71.

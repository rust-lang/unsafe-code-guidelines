When a crate explicitly acknowledges that what it does is UB, but prefers keeping that code over UB-free alternatives (or there are no UB-free alternatives), that is always a concerning sign.
We should evaluate whether there truly is some use-case here that is not currently achievable in well-defined Rust, and work with crate authors on providing a UB-free solution.

## Known cases of deliberate UB

### Cases related to concurrency

* crossbeam's `AtomicCell` implements an SeqLock, which [is well-known to not be compatible with the C++ memory model](http://www.hpl.hp.com/techreports/2012/HPL-2012-68.pdf).
  Specifically the problem is [this non-atomic volatile read](https://github.com/crossbeam-rs/crossbeam/blob/5d07fe43540d7f21517a51813acd9332744e90cb/crossbeam-utils/src/atomic/atomic_cell.rs#L980) which can cause data races and hence UB.
  This would be fine if we either (a) adopted LLVM's handling of memory races (then the problematic read would merely return `undef` instead of UB due to a data race), or (b) added [bytewise atomic memcpy](https://github.com/rust-lang/rfcs/pull/3301) and used that instead of the non-atomic volatile load.
  This is currently *not* UB in the LLVM IR we generate, due to LLVM's different handling of read-write races and because the code carefully uses `MaybeUninit`.
* crossbeam's `AtomicCell` also uses the standard library `Atomic*` types to do atomic reads and writes of [*any type* that has the right size](https://github.com/crossbeam-rs/crossbeam/blob/master/crossbeam-utils/src/atomic/atomic_cell.rs#L928-L932).
  However, doing an `AtomicU32` atomic read on a `(u16, u8)` is unsound because the padding byte can be uninitialized.
  (Also, pointer provenance is lost.)
  To fix this we need to be able to perform atomic loads at type `MaybeUninit<u32>`.
  The LLVM IR we generate here contains `noundef` so this UB is not just a specification artifact.
* `bytes` does [a non-atomic plain load that races](https://github.com/tokio-rs/bytes/blob/dea868a4b0eec28877e9013702c0ae12dbc40c4b/src/bytes.rs#L2508),
  because relaxed loads are claimed to cost too much performance (also see [this LLVM issue](https://github.com/llvm/llvm-project/issues/37064)).
  (Note that LLVM's handling of data races is not enough here, data races still return garbage data -- so this is UB in the generated LLVM IR. Also see [this thread](https://internals.rust-lang.org/t/unordered-as-a-solution-to-bit-wise-reasoning-for-atomic-accesses/11079) on using "unordered".)
  The best fix is probably to improve LLVM optimizations on relaxed loads, so that they are not considered too slow for this use-case.
  However, it is not clear if all the desired optimizations are even legal for relaxed loads -- there is a significant semantic difference between relaxed loads and non-atomic loads, so seeing *some* performance difference is expected.
  Code that works with concurrency can't just expect not to have to pay a cost for that.
* `smol-rs/event-listener` needs an SC fence, but on x86 they prefer a `lock cmpxchg` over an `mfence`.
  To achieve this, they do an [SC `compare_exchange` on a local variable](https://github.com/smol-rs/event-listener/blob/0ea464102e74219aab2932f9eff14418a13268d4/src/notify.rs#L574-L577).
  That is UB (both in Rust and LLVM) since such an operation has no synchronization effect; the compiler could easily see that this variable is never accessed from outside this function and hence optimize it away entirely.
  The best fix is probably an inline assembly block.

### Cases related to aliasing

* `yoke` and similar crates relying in "stable deref" properties cause various forms of aliasing trouble (such as [having `Box` that alias with things](https://github.com/unicode-org/icu4x/issues/2095), or [having references in function arguments that get deallocated while the function runs](https://github.com/unicode-org/icu4x/issues/3696)).
  This also violates the LLVM assumptions that come with `noalias` and `dereferenceable`.
  This could be fixed by [`MaybeDangling`](https://github.com/rust-lang/rfcs/pull/3336).
* The entire `async fn` ecosystem and every hand-implemented self-referential generator or future is unsound since the self-reference aliases the `&mut` reference to the full generator/future.
  This is currently hackfixed by making `Unpin` meaningful for UB; a proper solution would be to add something like [`UnsafeAliased`](https://github.com/rust-lang/rfcs/pull/3467).

## Former cases of deliberate UB that have at least a work-in-progress solution to them

* Various `offset_of` implementations caused UB by using `mem::uninitialized()`, or they used `&(*base).field` or `addr_of!((*base).field)` to project a dummy pointer to the field which is UB due to out-of-bounds pointer arithmetic.
  The `memoffset` crate has a sound implementation that however causes stack allocations which the compiler must optimize away.
  This will be fixed properly by the native [`offset_of!` macro](https://github.com/rust-lang/rfcs/pull/3308), which is [currently in nightly](https://github.com/rust-lang/rust/issues/106655).
* It used to be common to unwind out of `extern "C"` functions which is UB, see [this discussions](https://internals.rust-lang.org/t/unwinding-through-ffi-after-rust-1-33/9521).
  This is fixed by `extern "C-unwind"`, which is stable since Rust 1.71.

# Tree Borrows

**Note:** This document is not normative nor endorsed by the UCG WG.  It is maintained by @RalfJung and @JoJoDeveloping to reflect what is currently implemented in Miri.

This is not a guide! See the [Tree Borrows paper](https://plf.inf.ethz.ch/research/pldi25-tree-borrows.html) for more information.

Changes since publication of the paper:

* Interior-Mutable shared references are no longer treated like raw pointers, instead they use the new `Cell` permission. This permission allows all foreign and local accesses.
* Mirroring Stacked Borrows, structs which contain an UnsafeCell now have that UnsafeCell's position tracked more finely-grained. It is no longer sufficient to just have an UnsafeCell somewhere in a struct to mark this as being interior-mutable everywhere.
* The state machine was refactored to eliminate the `accessed` bit. This mostly affects the protected state machine. This changed no behavior but makes it less obvious how the current MiniRust implementation relates to the paper. See [this PR](https://github.com/minirust/minirust/pull/276) and also [this rendering of the new state machine(s)](https://github.com/user-attachments/files/23404334/figures2.pdf).

## MiniRust

Tree Borrows is fully documented in [MiniRust](https://github.com/minirust/minirust/tree/master/spec/mem/tree_borrows). MiniRust is written as literate code and should be readable without further explanation. The MiniRust version of Tree Borrows is the authoritative version, and it will be updated to reflect future changes. MiniRust defines all of Tree Borrows, including the more obscure features.

Instead of yet again defining Tree Borrows in prose here, we refer to MiniRust. The information below is not normative and only a summary of what is already explained in MiniRust.


### High-level summary

Tree Borrows maintains a tree for each allocation. Each pointer has a tag, that identifies a node in this tree.
Each node, for each offset/byte in the allocation, tracks a permission. The permission is per-byte, i.e. each byte has its own independent permission.
The permission evolves according to a state machine, which depends on the access (read/write), the relation between accessed and affected node (local/foreign), the current state, and whether the current node is protected by a protector.

The state machine differs for protected and unprotected nodes.
For protected nodes, some states additionally track whether this byte was "accessed" which formally means that the state exists twice (e.g. `Frozen` and `FrozenL`).
The protected state machine of Figure 3 in the paper only shows those that are considered accessed; we refer to the MiniRust implementation for the full details.


### Differences between MiniRust and Miri

MiniRust includes an idealized implementation of Tree Borrows, intended for easy readability.
In particular, it models provenance/tags as tree addresses, which uniquely identify a node in the borrow tree. Miri however uses unique integer IDs, with the Tree being tracked more implicitly as maps/relations between these IDs. The precise implementation of the tree is an implementation detail and not relevant for the semantics.

Besides this representation difference, Miri also includes a number of optimizations that make Tree Borrows have acceptable performance. These include:
* skipping nodes based on past foreign accesses, exploiting idempotence properties in the state machine
* garbage collection of unused references, which allows shrinking trees
* skipping nodes based on the permissions found therein

Additionally, the above-mentioned "refactor to eliminate the `accessed` bit" has not yet happened in Miri.

Furthermore, Miri now has experimental support for `-Zmiri-permissive-provenance` with Tree Borrows.

## Concepts Inherited From Stacked Borrows

### Retags

Tree Borrows has retags happen in the same place as Stacked Borrows. But note that Tree Borrows treats raw pointer retags as NOPs, i.e. it does not distinguish a raw pointer from the references it is derived from.

### Protectors

Like Stacked Borrows, Tree Borrows has protectors. These serve to ensure that references remain live throughout a function. Protectors come in "strong" and "weak" forms, as in SB, and they protect the same places in the same way.

### Implicit Reads and Writes

Like Stacked Borrows, Tree Borrows performs implicit accesses as part of retags. Unlike Stacked Borrows, these are always reads, even for `&mut` references. (But this might change, see below.)

A new concept in TB are implicit protector end accesses. These can be writes. See the section on "protector end semantics" in the paper for more info.

### UnsafeCell tracking

Like Stacked Borrows, Tree Borrows tracks where there are UnsafeCells, and treats these bytes differently from other bytes. UnsafeCells are tracked in structs and tuple fields, but enums are not inspected further.

### Accesses

Besides for the aforementioned differences in the handling of retags, what counted as a read or write in Stacked Borrows also counts as a read or write in Tree Borrows. These places are not surprising.

## Imprecisions

The following is a list of things that are _not_ UB in Tree Borrows. Some people want to make these things UB, so that more optimizations become possible. This is currently undecided and might just happen. In particular, all things listed here are already UB in Stacked Borrows.

* Tree Borrows does _not_ have subobject provenance, meaning that retags do not shrink the set of offsets that a reference can be used to access.
* Tree Borrows does not initially consider `&mut` references writable, it only does so after the first write. In practice, this might mean that optimizations moving writes up above the first write are forbidden.
  Note that there is work currently happening to make this a configurable option, and this option could become the default depending on the fallout of this change. See [#584](https://github.com/rust-lang/unsafe-code-guidelines/issues/584)

## Other problems

* The interaction of protector end writes with the data race model is not fully resolved. See [#585](https://github.com/rust-lang/unsafe-code-guidelines/issues/585).
* In Miri, there is experimental support for Tree Borros with `-Zmiri-permissive-provenance`, but the semantics are not yet fixed and likely to evolve in the future.

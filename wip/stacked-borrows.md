# Stacked Borrows

**Note:** This document is not normative nor endorsed by the UCG WG.  It is maintained by @RalfJung to reflect what is currently implemented in [Miri].

For more background on the how and why of Stacked Borrows, see this [blog post on the first implementation][stacked-borrows-1] and this [follow-up][stacked-borrows-2].

[Miri]: https://github.com/solson/miri/
[stacked-borrows-1]: https://www.ralfj.de/blog/2018/11/16/stacked-borrows-implementation.html
[stacked-borrows-2]: https://www.ralfj.de/blog/2018/12/26/stacked-borrows-barriers.html

## Extra state

Stacked Borrows adds some extra state to the Rust abstract machine.
Every pointer value has a *tag* (in addition to the location in memory that the pointer points to), and every memory location carries a *stack* (in addition to the byte of data stored at that location).
Moreover, there is a per-call-frame `CallId` as well as some global tracking state.

```rust
// `nat` is the type of mathematical natural numbers, meaning we don't want to think about overflow.
// NOTE: Miri just uses `u64` which, realistically, will not overflow because we only ever increment by 1.
type Timestamp = nat;

// Extra per-pointer state (the "tag")
pub enum Borrow {
    Uniq(Timestamp),
    Shr(Option<Timestamp>),
}

pub enum BorStackItem {
    Uniq(Timestamp),
    Shr,
    FnBarrier(CallId),
}
// Extra per-location state
pub struct Stack {
    borrows: Vec<BorStackItem>, // used as a stack; never empty
    frozen_since: Option<Timestamp>, // virtual frozen "item" on top of the stack
}

// Extra per-call-frame state
type CallId = nat;

// Extra global state
pub struct Tracking {
    clock: Timestamp,
    next_call: CallId,
}
```

These exist separately, i.e., when a pointer is stored in memory, then we both have a tag stored as part of this pointer value (remember, [bytes are more than `u8`](https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html)), and every byte occupied by the pointer has a stack regulating access to this location.
Also these two do not interact, i.e., when loading a pointer from memory, we just load the tag that was stored as part of this pointer.
The stack of a location, and the tag of a pointer stored at some location, do not have any effect on each other.

## Retag statement

Stacked Borrows introduces a new operation (a new MIR statement) on the Rust abstract machine.
*Retagging* operates on a place, and it also carries a flag indicating the kind of retag that is being performed:

```rust
pub enum RetagKind {
    /// The initial retag when entering a function
    FnEntry,
    /// Retag preparing for a two-phase borrow
    TwoPhase,
    /// Retagging raw pointers
    Raw,
    /// A "normal" retag
    Default,
}
```

`Retag` is inserted into the MIR for the following situations:

* A retag happens after every assignment MIR statement where the assigned type may contain a reference type (checking recursively below compound types but not below references).
  This is usually a `Default` retag.  However, if the RHS of this assignment is a `Ref` which allows two-phase borrows, then this is a `TwoPhase` retag.

    Currently, if the LHS of the assignment involves a `Deref`, no `Retag` is inserted.
    That's just a limitation of the current implementation: after executing this assignment, evaluating the place (the LHS) again could yield a different location in memory, which means we would retag the wrong thing.
    Proper retagging here requires either a copy through a temporary, or making retagging integral part of the semantics of assignment.

* A `Raw` retag happens after every assignment where the RHS is a cast from a reference to a raw pointer.

* A `FnEntry` retag happens in the first basic block of every function, retagging each argument.

* A `Default` retag happens on the return value of every function that gets called (i.e., this is the first statement in the basic block that the call will return to).

* The automatically generated drop shims perform a `Raw` retag of their argument because they use it as a raw pointer.

## Operational semantics

### Generating timestampts

Whenever we need to generate a new timestamp, that means we effectively call the following method:

```rust
impl Tracking {
    fn increment_clock(&mut self) -> Timestamp {
        let val = self.clock;
        self.clock += 1;
        val
    }
}
```

This method will never return the same value twice.

### Tracking function calls

To attach metadata to a particular function call, we assign a fresh ID to every call stack (so this distinguishes multiple calls to the same function).
In other words, the per-stack-frame `CallId` is initialized by the following method:

```rust
impl Tracking {
    fn new_call(&mut self) -> CallId {
        let id = self.next_call;
        self.next_call += 1;
        id
    }
}
```

This method will never return the same value twice.
We say that a `CallId` is *active* if the call stack contains a stack frame with that ID.

**Note**: Miri uses a slightly more complex system with a `HashSet<CallId>` tracking the set of active `CallId`; that is just an optimization to avoid having to scan the call stack all the time.

### Allocating memory

When allocating memory, we have to initialize the `Stack` associated with the new locations, and we have to choose a `Borrow` (a tag) for the initial pointer to this memory.

For most memory, the stack of each freshly allocated memory location is `Stack { borrows: vec![Shr], frozen_since: None }`, and the initial pointer to that memory has tag `Shr(None)`.

The only exception is stack memory.
Stack memory is handled by an environment (which is part of the information carried in a stack frame of the Rust abstract machine) that maps each local variable to a place.
A place is a pointer together with some other data that is not relevant here -- the key point is that a place, just like every other pointer, carries a tag.
When the local variable becomes live and its backing memory gets allocated, we generate a new timestamp `t` and use `Uniq(t)` as timestamp for the place of this local variable.
We also initialize the stack of all the memory locations in this new memory allocation with `Stack { borrows: vec![Uniq(t)], frozen_since: None }`.

### Accessing memory

On every memory access, we perform the following extra check for every location that gets accessed (i.e., for a 4-byte access, this happens for each of the 4 bytes):

1. If the location is frozen (`frozen_since.is_some()`) and this is a read access, nothing happens (even if the tag is `Uniq`).
2. Otherwise, if this is a write access, unfreeze the location (set `frozen_since` to `None`).
   (If this is a read access and we come here, the location is already unfrozen.)
3. Pop the stack until the top item matches the tag of the pointer.
    - A `Uniq` item matches a `Uniq` tag with the same ID.
    - A `Shr` item matches any `Shr` tag (with or without timestamp).
    - When we are reading, a `Shr` item matches a `Uniq` tag.
    - If we pop a `FnBarrier(c)` where `c` is active, we have undefined behavior.
    
    If we pop the entire stack without finding a match, then we have undefined behavior.

### Dereferencing a pointer

Every time a pointer gets dereferenced (evaluating the `Deref` place projection), we determine the extent of memory that this pointer covers using `size_of_val` and then we perform the following check on every location covered by the reference:

1. The location must exist, i.e., the pointer must actually be dereferencable for this entire memory range it covers.
2. If this is a raw pointer, stop here.  Raw accesses are checked as little as possible.
3. If this is a unique reference and the tag is `Shr(Some(_))`, that's an error.
4. If the tag is `Uniq`, make sure there is a matching `Uniq` item with the same ID on the stack.
5. If the tag is `Shr(None)`, make sure that either the location is frozen or else there is a `Shr` item on the stack.
6. If the tag is `Shr(Some(t))`, then the check depends on whether the location is inside an `UnsafeCell` or not, according to the type of the reference.
    - Locations outside `UnsafeCell` must have `frozen_since` set to `t` or an older timestamp.
    - `UnsafeCell` locations must either be frozen or else have a `Shr` item in their stack (same check as if the tag had no timestamp).

Whenever we are checking whether an item is in the stack, we ignore barriers.
Failing any of these checks means we have undefined behavior.

### Reborrowing

We define the notion of "reborrowing", which will be used below to define the semantics of `Retag`.
Reborrowing takes a (typed) place, whether to push a barrier, and the new borrow that this place is to be reborrowed for.
we determine the extent of memory that this place covers using `size_of_val` and then we perform the following actions on every location covered by the place:

1. Perform the checks that would also happen on a dereference.
   Remember the position of the item matching the tag in the stack.
2. Redundancy check, only happens if we will not push a barrier: if the new tag passes the checks performed on a dereference, and if the item that makes this check succeed is *above* the one we remembered in step 1 (where the "frozen" state is considered above every item in the stack), then stop.
   We are done for this location.
   This can only happen for shared references (i.e., when the borrow is `Shr(_)`).
3. Perform the actions that would also happen when an actual access happens through this reference (for shared references with borrow `Shr(_)` this is a read access, for mutable references with borrow `Uniq(_)` it is a write access).
    Now the location cannot be frozen any more: if the new borrow is `Uniq(_)`, we just unfroze; if it is `Shr(_)` and the location was already frozen, then the redundancy check (step 3) would have kicked in.
4. If we want to push a barrier, push `FnBarrier(c)` to the location stack where `c` is the `CallId` if the current function call (i.e., of the topmost frame in the call stack).
5. Check if the new tag is `Shr(Some(t))` and the location is inside an `UnsafeCell`.
    - If both conditions are satisfied, freeze the location with timestamp `t`.  If it is already frozen, do nothing.
    - Otherwise, push a new item onto the stack: `Shr` if the tag is a `Shr(_)`, `Uniq(id)` if the tag is `Uniq(id)`.

### Retagging

When executing `Retag(kind, place)`, we recursively visit all fields of this place, descending into compound types (`struct`, `enum`, arrays and so on) but not below any pointers.
For each reference (`&[mut] _`) and box (`Box<_>`) we encounter, and if `kind == Raw` also for each raw pointer (`*[const,mut] _`), we perform the following steps:

1. We compute a fresh tag: `Uniq(_)` for mutable references, `Box`, `Shr(Some(_))` for shared references, and `Shr(None)` for raw pointers.
2. We determine if we will want to push a barrier.
   This is the case only if all of the following conditions are satisfied: `kind == FnBarrier`, the type of this pointer is a reference (not a box), and if this is a shared reference then we are not inside an `UnsafeCell`.
3. We perform reborrowing with the new tag and indicating whether we ant a barrier pushed or not.
4. If `kind == TwoPhase`, we perform *another* reborrow with the tag being `Shr(Some(t))` for some fresh timestamp `t`, and not pushing new barriers.

### Deallocating memory

Memory deallocation first acts like a write access through the pointer used for deallocation.
After that is done, we additionally check all `FnBarrier(c)` occurring in any stack on any of the deallocated locations.
If any of the `c` is still active, we have undefined behavior.

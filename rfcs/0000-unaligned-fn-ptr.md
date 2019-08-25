- Feature Name: `unaligned_fn_ptr`
- Start Date: 2018-08-25
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Declare that function pointers do not make any alignment or other requirements to be "valid", beyond being non-null.

# Motivation
[motivation]: #motivation

Rust requires every value that is "produced" to be "valid" according to its type, even if the value is never actually "used" (see [Reference][ref-ub], [Nomicon][nom-ub]).
Unsafe code needs to know these rules to be sure it does not cause undefined behavior.
So what should the rules be for function pointers (`fn` types)?
They definitely must not be null, but is there anything else?
Possible options for additional requirements include alignment, and some kind of well-formedness of the data it points to ("it must be executable code", or something like that).
This is currently an open question.

The motivation to settle this question is that we have to say *something* in the Reference and the Nomicon that is at least a reasonable "upper bound" for the validity requirements of function pointers.
Otherwise, unsafe code creating function pointers is left without rules to follow.
With [this PR for the Reference](https://github.com/rust-lang-nursery/reference/pull/659), we have reasonable upper bounds for *most* types of Rust (these have not gone through a formal consensus process, but to my knowledge they form an upper bound for every proposal that has been made for validity of any type thus far).
The only exception is function pointers.
Saying anything about their alignment requirements is actually non-trivial: we would have to introduce the notion of a platform-dependent "function alignment", and then ask users to look that up for the platforms they are targeting.
Preferably, there would be way for code to actually query this alignment requirement (similar to `mem::align_of`).
To avoid having to create all that infrastructure, I propose we instead declare that function pointers do not have to be aligned.

On most current platforms, we could not make any alignment requirement anyway: on x86, there is no alignment for code, and while ARM does have 2-byte alignment, it uses the least significant bit to encode the instruction set (thumb or not) of the pointee, so function pointers themselves are not guaranteed to be aligned.

Beyond alignment, while we could require the function pointer to point to a "valid sequence of instructions", it is unclear what the benefit of that would be.

Hence, given the lack of a motivation *for* making any non-null function pointers invalid, and in order to simplify the rules unsafe code has to follow and remove needless footguns, we propose to specify null as the only value that is invalid for function pointers.

[ref-ub]: https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html
[nom-ub]: https://doc.rust-lang.org/nightly/nomicon/what-unsafe-does.html

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When transmuting data to a function pointer, you must make sure it is non-null:

```rust
fn bad() {
    let x: fn() = unsafe { mem::transmute(0usize) } // This is UB!
}
```

Transmuting any non-null value to a function pointer is defined behavior.

```rust
fn good() {
    let x: fn() = unsafe { mem::transmute(1usize) } // This is not UB.
}
```

Further requirements, outside the scope of this RFC, come into play when the function pointer gets called.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A function pointer is "valid" (in the sense that it can be produced without causing immediate UB) if and only if it is non-null.

# Drawbacks
[drawbacks]: #drawbacks

This closes the door to potential optimizations based on more invalid values for function pointers.
For example, if function pointers had to be 2-aligned, `Option<Option<fn()>>` could be pointer-sized because there is a niche of size 2 in `fn()`.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The rationale was given further up in the motivation.

The alternative is to require function pointers to be aligned to some platform-specific quantity.
In that case we should specify what that quantity is for our platforms.

# Prior art
[prior-art]: #prior-art

To our knowledge, no other language makes it undefined behavior to merely *create* (but not use) an unaligned or otherwise "bad" function pointer.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None we can think of.

# Future possibilities
[future-possibilities]: #future-possibilities

What exactly are the rules for *calling* a function pointer?
This is deliberately out of scope of this RFC.

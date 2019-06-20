# Validity of unions

**Disclaimer**: This chapter is a work-in-progress. What's contained here
represents the consensus from issue [#73]. The statements in here are not (yet)
"guaranteed" not to change until an RFC ratifies them.

The bit `i` of an union is allowed to have value `v` _iff_ there is a variant of
the union such that bit `i` of the variant is allowed to have value `v`. We
assume all variants to be "filled up" to the same size with padding, which may
have any value.

## Validity of unions with zero-sized fields

A union containing a zero-sized field can contain any value. An example of such
an union is [`MaybeUninit`].

<details><summary><b>Rationale</b></summary> 

This follows from the definition of the values that `union` bits are allowed to
take. The zero-sized type has size `0`, and its variant is filled up to
the `union` size with padding. Since padding is allowed to take any value, any
bit in the `union` is allowed to take any value.

[#73]: https://github.com/rust-lang/unsafe-code-guidelines/issues/73
[`MaybeUninit`]: https://doc.rust-lang.org/std/mem/union.MaybeUninit.html

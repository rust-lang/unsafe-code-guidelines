# Validity of unions

**Disclaimer**: This chapter is a work-in-progress. What's contained here
represents the consensus from issue [#73]. The statements in here are not (yet)
"guaranteed" not to change until an RFC ratifies them.

## Validity of unions with zero-sized fields

A union containing a zero-sized field can contain any bit pattern. An example of such
an union is [`MaybeUninit`].

[#73]: https://github.com/rust-lang/unsafe-code-guidelines/issues/73
[`MaybeUninit`]: https://doc.rust-lang.org/std/mem/union.MaybeUninit.html

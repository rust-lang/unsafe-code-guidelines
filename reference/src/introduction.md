## Rust's Unsafe Code Guidelines Reference

This document is produced by the [UCG WG][ucg_wg] to provide a "guide" for
writing unsafe code that "recommends" what kinds of things unsafe code can and
cannot do, and that documents which guarantees unsafe code may rely on. It is
largely a work-in-progress right now.

Unless we state otherwise, the information in the guide is mostly a
"recommendation" and still subject to change. The main goal for the WG in 2019
is to RFC parts of the guide into the language specification, and move those
parts into the [Rust Language Reference] proper.

[ucg_wg]: https://github.com/rust-lang/unsafe-code-guidelines
[Rust Language Reference]: https://doc.rust-lang.org/reference/index.html

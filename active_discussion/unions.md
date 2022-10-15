# Unions

## Outstanding questions

* Is `#[repr(Rust)]` the bag-o-bytes union repr, or do we want to propose a new repr?
    * *Discussion:* [#73: Validity of unions][#73]
* The following questions are all implicitly answered if `#[repr(Rust)]` is the bag-o-bytes repr, but remain open if not:
    * Do `#[repr(Rust)]` unions guarantee all fields at offset 0?
        * *Discussion*: [#353: Offsets of union fields][#353]
    * Do `#[repr(Rust)]` unions have internal padding?
        * *Discussion*: [#354: Do #[repr(Rust)] unions have internal padding?][#354]
* Do `#[repr(transparent)]` unions ever have niches?
    * *Discussion*: [#364: What is the value model/validity invariant for transparent unions?][#364]

## Closed discussion issues:

* [#13: Representation of unions][#13]
* [#156: Layout of repr(C) unions has padding][#156]
* [#298: Is `repr(transparent)` completely transparent within `repr(Rust)` types?][#298]
* [#352: What is the safety invariant, if any, for unions?][#352]

[#13]: https://github.com/rust-lang/unsafe-code-guidelines/issues/13
[#156]: https://github.com/rust-lang/unsafe-code-guidelines/issues/156
[#298]: https://github.com/rust-lang/unsafe-code-guidelines/issues/298
[#352]: https://github.com/rust-lang/unsafe-code-guidelines/issues/352
[#353]: https://github.com/rust-lang/unsafe-code-guidelines/issues/353
[#354]: https://github.com/rust-lang/unsafe-code-guidelines/issues/354
[#364]: https://github.com/rust-lang/unsafe-code-guidelines/issues/364
[#73]: https://github.com/rust-lang/unsafe-code-guidelines/issues/73

UCG - Rust's Unsafe Code Guidelines
===

The purpose of this repository is to collect and discuss all sorts of questions that come up when writing unsafe code.
It is primarily used by the [opsem team](https://github.com/rust-lang/opsem-team/) to track open questions around the operational semantics, but we also track some "non-opsem" questions that fall into T-lang or T-type's purview, if they are highly relevant to unsafe code authors.

The [Unsafe Code Guidelines Reference "book"][ucg_book] is a past effort to systematize a consensus on some of these questions.
Most of it has been archived, but the [glossary](https://rust-lang.github.io/unsafe-code-guidelines/glossary.html) is still a useful resource.

Current consensus is documented in [t-opsem FCPs](https://github.com/rust-lang/opsem-team/blob/main/fcps.md) and the [Rust Language Reference].

[ucg_book]: https://rust-lang.github.io/unsafe-code-guidelines
[Rust Language Reference]: https://doc.rust-lang.org/reference/index.html

## See also

The [Rustonomicon] is a draft document discussing unsafe code. It is intended to
be brought into agreement with the content here. It represents an organized
effort to explain how to write Rust code, rather than a reference.

[Rustonomicon]: https://doc.rust-lang.org/nightly/nomicon/

## Code of Conduct and licensing

All interactions on this repository (whether on issues, PRs, or
elsewhere) are governed by the [Rust Code of
Conduct](CODE_OF_CONDUCT.md).

Further, all content on this repository is subject to the standard
[Rust](LICENSE-MIT) [licensing](LICENSE-APACHE).

[![Travis-CI Status]][travis] [![UCG Book]][ucg_book] 

[travis]: https://travis-ci.com/rust-lang/unsafe-code-guidelines
[Travis-CI Status]: https://travis-ci.com/rust-lang/unsafe-code-guidelines.svg?branch=master
[UCG Book]: https://img.shields.io/badge/UCG%20Book-Unsafe%20Code%20Guidelines%20Book-blue.svg
[ucg_book]: https://rust-lang.github.io/unsafe-code-guidelines

UCG WG - Rust's Unsafe Code Guidelines Working Group
===

The goal of the UCG WG is to produce the [Unsafe Code Guidelines
Reference][ucg_book]: a "guide" for writing unsafe code that "recommends" what
kinds of things unsafe code can and cannot do. It is largely a work-in-progress
right now.

Unless we state otherwise, the information in the guide is mostly a
"recommendation" and still subject to change. The main goal for the WG in 2019
is to RFC parts of the guide into the language specification, and move those
parts into the [Rust Language Reference] proper.

[Rust Language Reference]: https://doc.rust-lang.org/reference/index.html

> **note**: the instructions to build the UCG reference book are in the
> [`.travis.yml`][travis_yml] file.

[travis_yml]: .travis.yml

## Process 

A more complete description of our process is available in [the `process.md`
file][process]. Briefly:

[process]: process.md

* **active discussion area**: At any given time, the UCG working group is
  focused on one particular **area** of discussion. The current area is
  [**Validity Invariants**][current_area] -- read that link for more background
  and information. You can find the [active discussion threads
  here][active_discussion].

[current_area]: active_discussion/validity.md
[active_discussion]: https://github.com/rust-lang/unsafe-code-guidelines/issues?q=is%3Aissue+is%3Aopen+label%3A%22active+discussion+topic%22

* **communication channels**: Most of the discussion takes place here in GitHub
  issues. Many of us do also hang out on [Zulip's `wg-unsafe-code-guidelines`
  stream][Zulip]. There is rust-lang org team called
  [`WG-unsafe-code-guidelines`][rust_team] that is commonly cc'd when curious
  situations arise. Send a private message to @nikomatsakis on [internals] or
  [Zulip] if you would like to be added (note: membership does not imply any
  particular decision-making power).

[Zulip]: https://rust-lang.zulipchat.com/#narrow/stream/136281-wg-unsafe-code-guidelines
[rust_team]: https://github.com/orgs/rust-lang/teams/wg-unsafe-code-guidelines
[internals]: https://internals.rust-lang.org/

* **meetings**: Every Thursday at 12:15 UTC-4 (16:15 UTC), we hold a brief
  _non-technical_ meeting on [Zulip] to triage issues, PRs, assign work, etc.
  Everybody is welcome to attend.

Ideas for future "areas of discussion" are represented as [issues labeled with
the "proposed discussion topic" label][proposed_discussion]. Feel free to open
more such issues if you have a question that doesn't match any of the existing
issues!

[proposed_discussion]: https://github.com/rust-rfcs/unsafe-code-guidelines/labels/proposed%20discussion%20topic

We follow the [Rust's Code of Conduct]. Any concerns should be addressed to the
[moderation team] immediately.

[Rust's Code of Conduct]: https://www.rust-lang.org/en-US/conduct.html
[moderation team]: https://www.rust-lang.org/team.html#Moderation-team

## See also

The [Rustonomicon] is a draft document discussing unsafe code. It is intended to
be brought into agreement with the content here. It represents an organized
effort to explain how to write Rust code, rather than a reference.

[Rustonomicon]: https://doc.rust-lang.org/nightly/nomicon/

The [nikomatsakis/rust-memory-model] was a previous effort and contains a lot of
good links and information.

[nikomatsakis/rust-memory-model]: https://github.com/nikomatsakis/rust-memory-model

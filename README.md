# unsafe-code-guidelines

Home for the Unsafe Code Guidelines (UCG) effort. The goal of the Unsafe
Code Guidelines effort is to collaboratively produce a "reference
guide" for writing unsafe code that what kinds of things unsafe code
can and cannot do.

## Active area of discussion

At any given time, the UCG working group is focused on one particular
**area**. The current area is [**To Be Determined**][currentarea] --
read that link for more background and information. You can find the
[current discussion threads here][threads].

Ideas for future "areas of discussion" are represented as [issues
labeled with the "proposed discussion topic"
label](https://github.com/rust-rfcs/unsafe-code-guidelines/labels/proposed%20discussion%20topic). Feel
free to open more such issues if you have a question that doesn't
match any of the existing issues!

We follow the [Rust code of
conduct](https://www.rust-lang.org/en-US/conduct.html).  Any concerns
should be addressed to [the moderation
team](https://www.rust-lang.org/team.html#Moderation-team)
immediately.

[currentarea]: active_discussion/TEMPLATE.md
[threads]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues?q=is%3Aopen+is%3Aissue+label%3Atopic-repr

For people who are interested in this sort of thing, there is a team
in the rust-lang org called `WG-unsafe-code-guidelines`. Send a
private message to @nikomatsakis on [internals] if you would like to
be added. This team is commonly cc'd when curious situations arise.
Being a member of this GitHub team simply indicates that you would
like to be notified and does not imply any particular decision making
power.

[internals]: https://internals.rust-lang.org/

## The "Unsafe Code Guidelines Reference"

Results from past discussions are written up in the "Unsafe Code
Guidelines Reference", which lives in this repository. It is largely a
work-in-progress right now. Note that, unless we state otherwise, the
information in there is mostly a "recommendation" and still subject to
change -- once made official, the intention is to migrate this
information into the Rust Reference proper. [Read the "Unsafe Code
Guidelines Reference" here.][rr]

[rr]: https://github.com/rust-rfcs/unsafe-code-guidelines/blob/master/reference/src/SUMMARY.md

## Chat platform, discussion cadence

Most of the discussion takes place here in GitHub issues. Many of us
do hangout on [Zulip], however, in the `wg-unsafe-code-guidelines`
stream.

[Zulip]: https://rust-lang.zulipchat.com/#narrow/stream/136281-wg-unsafe-code-guidelines

Every two weeks, we hold a brief, non-technical meeting to decide
whether to keep the current area of discussion or to adopt a new
one. Prior to this meeting, we update the "area of discussion"
document with summaries of the threads. The meeting takes place on
[Zulip] and all are welcome to attend.

A more complete description of our process can be found in [the `process.md` file][p].

[p]: 

## See also

The [Rustonomicon](https://doc.rust-lang.org/nightly/nomicon/) is a
draft document discussing unsafe code. It is intended to be brought
into agreement with the content here. It represents an organized
effort to explain how to write Rust code, rather than a reference.

[The nikomatsakis/rust-memory-model
repository](https://github.com/nikomatsakis/rust-memory-model) was a
previous effort and contains a lot of good links and info.

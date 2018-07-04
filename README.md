# unsafe-code-guidelines

Home for the Unsafe Code Guidelines effort. The goal of the Unsafe
Code Guidelines effort is to collaboratively produce a "reference
guide" to write unsafe code. This guide should ultimately be a formal
specification of what kinds of things unsafe code can and cannot do.
However, we are not there yet.

## The reference

The primary document in this repo is the "Unsafe Code Guidelines
Reference".  It is largely a work-in-progress right now. It serves as
a kind of table of contents to the sorts of questions that are on our
radar, as well as place to collect notes and summaries about those
questions.

Since we are not yet at the point where we can provide complete,
definitive answers, the current goal for each section is to describe
the "contours of the space". For example, we would try to specify what
kinds of invariants **must** hold, as well as things that we expect to
**never** be true. For other cases where there is disagreement, we
will try to summarize the various options on the table -- you would do
well in your unsafe code to steer clear of relying on those details.

## How to participate

The Unsafe Code Guidelines WG is in the process of being rebooted.  We
expect to be holding regular meetings focused on particular topics.
Stay tuned for more announcements. -- @nikomatsakis



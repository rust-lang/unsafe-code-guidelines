### TL;DR

The basic idea is simple: we try to focus discussion on one area at a
time. The discussion begins with a proposal outlining the goals ([here
is an example of such a proposal][eg]). From there, we open up various
threads that are discussing "mostly independent" topics. Over the
course of the discussion, a "discussion lead" will periodically
collate the details into a document living in the
[unsafe-code-guidelines repository][repo], so that people can get up
to speed more quickly. At some point, when we feel like we've gone as
far as we can go, we "wrap up" the discussion and select another thing
to talk about.

[repo]: https://github.com/rust-rfcs/unsafe-code-guidelines/

### Starting a discussion

To start a discussion, a lead (or group of leads) prepares a
"discussion proposal". This has the form of a PR. The basis for the
proposal is [the `TEMPLATE.md` file][template]. The idea is that this
proposal will:

[template]: active_discussion/TEMPLATE.md

- outline the topic to be discussed and the goals of the discussion, 
- highlight some interesting questions and background reading, 
- and propose an initial set of threads.

On the PR, we can discuss whether this is a good thing to talk about
and what changes we might make. For example, maybe we want to narrow
the topic, or add some more examples.

### Repo and Threads

Once we decide to adopt an area for discussion (see "meetings" below),
we do the following:

- merge the PR and update the README.md appropriately
- create a `topic-foo` label for this topic
- open the initial discussion threads, tagging them with the new label

### Ending a discussion

At some point, we will decide in one of the meetings (see below) that
the discussion has reached a steady state (no new points are being
made). The leads can then take the final summaries and move that
content into our "reference", which basically collates all the results
from prior discussions. Note that discussions don't have to reach firm
conclusions: it is ok to end with "these things are known" as well as
"these things need to be settled in the future".

### Regular meetings and schedule

In general, regular meetings are held every two weeks (the time and
venue appears in the README). Meetings are used to decide when to change
topics and so forth. 

Attendance at the meeting is welcome but not mandatory or
expected. Prior to the meeting itself, the discussion leads will open
a PR with updated summaries and a suggestion of whether to move on
etc. It is suggested that you comment on this PR.

### Making things official

Conclusions reached by this working group are effectively
"recommendations".  To become official, they have to go through the
Rust RFC process as a lang team RFC.

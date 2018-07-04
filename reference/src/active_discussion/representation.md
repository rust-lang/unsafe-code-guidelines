# Data structure representation

In general, Rust makes few guarantees about memory layout, unless you
define your structs as `#[repr(rust)]`. But there are some things that
we do guarantee. Let's write about them.

TODO:

- Find and link to the various RFCs
- Enumerate things that we *might* in fact guarantee, even for non-C types:
  - e.g., `&T` and `Option<&T>` are both pointer sized
  - size of `extern fn` etc (at least on some platforms)?


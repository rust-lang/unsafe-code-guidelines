# Layout requirements of the C ABI

**Disclaimer:** This chapter is a minimal summary of issue [#44] about Rust's
current layout requirements of the platforms' C ABI.

These requirements apply to all C ABIs that Rust programs interface with, e.g.,
via `extern "C"`. That is, for Rust to be able to target a platform, this
platform does not have to provide a C implementation, but then all Rust programs
attempting to interface with one are illegal.

The platform's C implementation is required to conform to the [latest version of
the C standard]. The following are extra requirements that Rust imposes on top
of the latest version of the C standard, e.g., on implementation-defined or
unspecified behavior:

* `CHAR_BITS == 8`
* `sizeof(Bool_) == 1`
* `true = 1` and `false = 0`
* two's complement integers
* have, at least, one pointer value that is never dereferenceable

## Unresolved questions

* Shall Rust require that the C's platform pointer size must be at lest, e.g.,
  16-bits wide?
* Does the pointer with representation `intptr_t(0)` have to never be
  dereferenceable?
* Are `float` and `double` optional? What should we require of these types
  (IEEE754:2018 compatibility?)? See [#9].
* Is atomics support optional?
  
[latest_c_std]: http://www.open-std.org/jtc1/sc22/wg14/www/abq/c17_updated_proposed_fdis.pdf

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
* `sizeof(_Bool) == 1`
* `true = 1` and `false = 0`
* two's complement integers

## Unresolved questions

* Shall Rust require that the C's platform pointer size must be at lest, e.g.,
  16-bits wide?
* Does the pointer with representation `intptr_t(0)` have to never be
  dereferenceable?
* Are `float` and `double` optional? What should we require of these types
  (IEEE754:2018 compatibility?)? See [#9].
* Is atomics support optional?

 * Once we start discussing validity, the bit pattern representing the `None`
   value of `Option<&T>` has to be specified. In particular, we want to require
   that this bitpattern matches that of a C pointer for which `ptr == NULL`
   returns `true`, to ensure that C pointers passed to Rust via FFI that are
   null also result in null pointers in Rust, `Option<&T>::None`, etc.
   
   If we decide that this null bit-pattern can only be `0x0`, then we'd have to
   add this requirement to this document in the future.
  
[latest_c_std]: http://www.open-std.org/jtc1/sc22/wg14/www/abq/c17_updated_proposed_fdis.pdf

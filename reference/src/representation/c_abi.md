# Layout requirements of the C ABI

**Disclaimer:** This chapter is a minimal summary of issue [#44] about Rust's
current layout requirements of the platforms' C ABI.

Rust programs can target platforms that do not have a C implementation. However,
when targeting these platforms, attempting to interface with C, e.g., via
`extern "C"` is illegal, since there is no C ABI to interface with. 

Interfacing with C is allowed in platforms that have a C implementation that
satisfies all requirements specified in this document.

## Requirements on the C implementation 

If the platform has a C implementation that can be interfaced with via C FFI,
this implementation is required to conform to the [latest version of the C
standard]. The following are extra requirements that Rust imposes on top of the
latest version of the C standard, e.g., on implementation-defined and/or
unspecified behavior:

* `CHAR_BITS == 8`
* `sizeof(_Bool) == 1`
* `true = 1` and `false = 0`
* two's complement integers

## Unresolved questions

The unresolved questions that don't contain a link to an issue have no issue
open yet. This document shall be updated as issues are resolved and new issues
are opened:

* **minimum platform pointer size**: should Rust require that the C's platform
  pointer size must be at lest, e.g., 16-bits wide?

* **float types** (32 and 64-bit wide only for now): see [#9]

  * Are these optional? 
  
  * Should we require anything about them beyond what the C standard requires?
  
  * These probably need to be IEEE754:2018 compliant and we should specify their
  binary representation. 
  
  * Do we need to specify anything else on top of IEEE754:2018, e.g., with
    respect to implementation-defined behavior?

* **atomics**:

  * Are these optional?
  
  * Should we require anything about them beyond what the C standard requires?

* **`NULL` pointer bitpattern**:

  * Does the C standard allow platforms without a run-time `NULL` representation?
  
  * Some platforms do not have any pointer value that compares equal to `NULL`.
    Should we support these platforms? For example, an OS kernel like Linux that
    wants to use the `0x0` address. Rust is used for these platforms already.

  * Some platforms have pointer values that compare equal to `NULL` but whose
    bit representation is not `0x0`. Should we support these platforms?
    * That is, should we only support C platforms where the only pointer
      representation that compares equal to `NULL` is `(void*)intptr_t(0)` ?

    * This interacts which the bit-patterns:
     
       * for which `&T` and `&mut T` have niches, and therefore `Option<&T>` is
         `None`
       * `<*[mut,const] T>::is_null(self)` returns `true` - note that `NonNull`
         does not convey any information about the exact representation of its
         niche, that is, it is different from `NonZero`.
  
[latest_c_std]: http://www.open-std.org/jtc1/sc22/wg14/www/abq/c17_updated_proposed_fdis.pdf

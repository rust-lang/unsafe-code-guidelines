# Keeping secrets

When storing cryptographic keys, crypto code wants to be sure that the
compiler will not insert loads or stores that were not present in the
source. Moreover, it wants to be able to zero memory and know that no
bits from that memory "escape" into registers etc.

## See also

- https://internals.rust-lang.org/t/volatile-and-sensitive-memory/3188
- https://github.com/nikomatsakis/rust-memory-model/issues/16

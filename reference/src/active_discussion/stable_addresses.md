# Stable addresses

Clearly, if you have a `&T` reference, the actual pointer address of
that memory must remain valid while **that reference** is in active
use. But how stable are the memory addresses of local variables in
between borrows? Consider:

```rust
let x = 22;
foo(&x);
foo(&x);

fn foo(y: &usize) { .. }
```

Is `foo` guaranteed to be given the same pointer each time? Note that
safe code can observe the pointer value by doing `y as *const usize as
usize`. If, however, the answer is no, that is helpful to the compiler
since it can spill `x` only while it is borrowed but otherwise simply
store `x` in a register.

**Range of possible answers:**

- local variables have stable addresses (de facto true today)
- local variables have stable addresses while borrowed, but may change betwen borrows (would be nice)

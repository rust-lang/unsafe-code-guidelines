## Glossary

#### Interior mutability

*Interior Mutability* means mutating memory where there also exists a live shared reference immediately (i.e., non-transitively) pointing to the same memory.
This propagates recursively through references, but not through raw pointers.
So, for example, if data immediately pointed to by a `&T` or `& &mut T` is mutated, that's interior mutability.
If data immediately pointed to by a `*const T` or `&*const T` is mutated, that's *not* interior mutability.

"live" here means a value that will be "used again" later.
This is not yet precisely defined, this will be fixed as part of developing a precise aliasing model.

Interior mutability is only allowed inside [`UnsafeCell`](https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html).

#### Validity and safety invariant

The *validity invariant* is an invariant that all data must uphold any time it is accessed or copied in a typed manner.
This invariant is known to the compiler and exploited by optimizations such as improved enum layout or eliding in-bounds checks.

In terms of MIR statements, "accessed or copied" means whenever an assignment statement is executed.
That statement has a type (LHS and RHS must have the same type), and the data being assigned must be valid at that type.
Moreover, arguments passed to a function must be valid at the type given in the callee signature, and the return value of a function must be valid at the type given in the caller signature.
OPEN QUESTION: Are there more cases where data must be valid?

In terms of code, some data computed by `TERM` is valid at type `T` if and only if the following program does not have UB:
```rust
fn main() { unsafe {
  let t: T = std::mem::transmute(TERM);
} }
```

The *safety* invariant is an invariant that safe code may assume all data to uphold.
This invariant is used to justify which operations safe code can perform.
The safety invariant can be temporarily violated by unsafe code, but must always be upheld when interfacing with unknown safe code.
It is not relevant when arguing whether some *program* has UB, but it is relevant when arguing whether some code safely encapsulates its unsafety -- IOW, it is relevant when arguing whether some *library* can be used by safe code to *cause* UB.

In terms of code, some data computed by `TERM` (possibly constructed from some `arguments` that can be *assumed* to satisfy the safety invariant) is valid at type `T` if and only if the following library function can be safely exposed to arbitrary (safe) code as part of the public library interface:
```rust
pub fn make_something(arguments: U) -> T { unsafe {
  std::mem::transmute(TERM)
} }
```

One example of valid-but-unsafe data is a `&str` or `String` that's not well-formed UTF-8: the compiler will not run its own optimizations that would cause any trouble here, so unsafe code may temporarily violate the invariant that strings are `UTF-8`.
However, functions on `&str`/`String` may assume the string to be `UTF-8`, meaning they may cause UB if the string is *not* `UTF-8`.
This means that unsafe code violating the UTF-8 invariant must not perform string operations (it may operate on the data as a byte slice though), or else it risks UB.
Moreover, such unsafe code must not return a non-UTF-8 string to the "outside" of its safe abstraction boundary, because that would mean safe code could cause UB by doing `bad_function().chars().count()`.

To summarize: *Data must always be valid, but it only must be safe in safe code.*
For some more information, see [this blog post](https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html).

#### Layout / Representation

The *layout* of a type defines its size and alignment as well as the offset of its subobjects, i.e., its fields/elements.
Moreover, the layout of a type records its *function call ABI* (or just *ABI* for short): how the type is passed *by value* across a function boundary.

We often use *representation* as a synonym for "layout".

### TODO

* *niche*
* *tag*
* *rvalue*
* *lvalue*

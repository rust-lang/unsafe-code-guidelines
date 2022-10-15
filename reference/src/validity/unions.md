# Validity of unions

**Disclaimer**: This chapter is a work-in-progress.
What's contained here represents the consensus from [various issues][union
discussion].
The statements in here are not (yet) "guaranteed" not to change until an RFC
ratifies them.

[union discussion]: https://github.com/rust-lang/unsafe-code-guidelines/blob/master/active_discussion/unions.md

**Note**: For ease of reading the examples, the hypothetical type `Padded<T>` is
used, which behaves identically to `T` except that writing to it clobbers any
subsequent padding.
This could actually be accomplished using an overaligned newtype struct, but
that would make the examples harder to read.
Additionally, layout of union fields is put in some comments; in these comments,
the notation `n*pad` means "`n` padding bytes".

## Value model

The possible values of unions are not defined in terms of the values of the
fields, but rather, a union's possible values are lists of bytes.
The [representation relation] is trivial in both directions, except for [padding
bytes] which are uninitialized in all values.

<details><summary><b>Rationale</b></summary>

The following examples must be supported, and therefore impose constraints on union value behaviour. The simplest solution, by far, is to treat the value representation of unions as merely being lists of bytes. While we do not discuss every possible angle here, it should be rapidly clear from just these two examples that any other example is significantly more complicated. Please trust us that every other alternatives that sounds even half-reasonable has been examined and has some critical flaw or another.

**We cannot require all fields, or even only one field, to be valid at all times.**

```rust
union Padding {
    left: (Padded<u8>, u16), // [[u8, 1*pad], u16]
    right: (u16, Padded<u8>), // [u16, [u8, 1*pad]]
}
let p = Padding{left: (0, 0)}; // resulting bytes: [0, uninit, 0, 0]
p.right.1 = 1; // resulting bytes: [0, uninit, 1, uninit]
fn f(_: Padding) {}
f(p);
```

By the end of this example, the resulting union has no valid fields, because every field contains uninit, non-padding bytes. So therefore no field is even fully initialized, despite the fact that `p` was fully initialized as can be witnessed by the fact that the compiler allows it to be moved into `f`. This is all stable, Safe Rust, so these semantics being sound are a hard constraint.

**Unions must preserve provenance.**

```rust
union Provenance<'a> {
    raw: *const u32,
    reference: &'a u32,
}
let x: u32;
let u = Provenance(raw: &x);
let y = unsafe{ *u.reference };
```

We must be able to carry provenance between the `raw` and `reference` fields in order for the assignment via `u.reference` to be valid. While this uses Unsafe Rust, this code is "obviously" sound. And therefore the union must be able to maintain provenance between the two pointers---even should the pointers be nested deeply within structs.

[representation relation]: ../glossary.md#representation-relation
[padding bytes]: ../glossary.md#padding-byte

</details>

### Niches

## Value space for #[repr(C)] and Raw-repr unions

`#[repr(C)]` and [Raw-repr][raw repr] unions can store any byte value, except for [padding bytes] which are always uninitialized.

<details><summary><b>Rationale</b></summary>

The purpose of the Raw-repr is to provide these semantics, which are easy to reason about. Furthermore, C programmers are used to being able to treat unions like bags of bytes, more or less, and Rust programmers are similarly used to the same with `#[repr(C)]` unions. Therefore, they should both accept any arbitrary byte pattern.

</details>

## Possible niche values

The presence of [padding bytes], and writes to individual fields in general, makes niches hard to come by in unions. A niche representation of a union would have to not only be invalid for every single one of its fields, but also impossible to construct in Safe Rust with any combination of writes to any of its fields.

For reprs other than `#[repr(C)]` and the [Raw-repr], values not constructible from safe Rust are consequently [**TBD**][#73] whether or not they are valid. The following example assumes that `#[repr(Rust)]` is not the Raw-repr:

```rust
#[repr(Rust)]
union U {
    a: (u16, u16),
    b: u32,
}
MaybeUninit<U>::uninit().assume_init(); // Unsound: assumes that U can be uninit
fn get_b(u: U) -> u32 {
    unsafe { u.b } // Unsound: assumes that U cannot be uninit
}
let u: U;
get_b(u); // Compile error: u is not initialized.
```

Because all bytes of `U` must be initialized for the value to be valid, and this is enforced by the compiler's initialization checks, it might be tempting to assume that `U`'s bytes must always be defined, but this is not a valid assumption. It is equally invalid, however, to assume that `U`'s bytes can be undefined.

<details><summary><b>Rationale</b></summary>

We have not yet reached consensus on whether or not we wish to leave the door open for the possibility that unions with safe field access, or `#[repr(transparent)]` unions with no ZSTs, contain niches:

```rust
#[repr(transparent)]
union U { b: bool };
assert_eq!(size_of::<Option<U>>(), 1); // Requires a niche, which in turn requires that U must be initialized to be valid.
```

We are **not** describing this case as unspecified, but instead as TBD."Constructible with Safe Rust" is a poorly-defined and very complex invariant, which falls short of the UCG's goals of easily checked, easy to understand (such as it were) semantics, and therefore we are not comfortable leaving the language in this state on an indefinite basis.

The main saving grace here is that `#[repr(Rust)]` unions are presently nearly impossible to use correctly anyways, because they do not even guarantee fields at offset 0.

</details>

## Validity of sometimes-padding bytes

We can say that a byte is *sometimes padding* for a union `U` if there is *some* inhabited field `f` such that the byte is either padding for `f` or not a part of `f`.

In that case, the byte will be uninitialized in the value `U{f: /* some value */ }`. By the [monotonicity property], therefore, all sometimes-padding bytes can contain any byte value, be it undefined or any bit pattern with any provenance. Likewise, if multiple bytes are padding for the same field, then they can take on any possible combination of byte values between them.

It follows that a union containing an inhabited zero-sized field can contain any bit pattern whatsoever, because all bytes are sometimes-padding bytes. An example of such
an union is [`MaybeUninit<T>`], which is a union of `T` and `()`.

As per the previous section, however, just because a byte is a sometimes-padding byte does not mean it can always safely be set to uninitialized (or any other value), if this can produce a value not reachable from Safe Rust.

For instance, the following is presently unsound (assuming that `#[repr(Rust)]` is not the [Raw-repr]), even assuming that all fields are placed at offset 0:

```rust
struct B {

}
#[repr(Safe)]
Union u {
    b: bool, // [bool, 1*pad]
    u: u16, // [u16]
}
let u = U{u: (0xff00, 0}; // resulting bytes: [0xff, 0]
unsafe { (&mut u.b as *mut u8 as *mut MaybeUninit<u8>).offset(1).write(MaybeUninit::uninit()) }; // resulting bytes: [0xff, uninit]
```

This value is impossible to reach in Safe Rust: the only way to write uninit to the padding is to write to the boolean field. Writing to the integer field must initialize 

## Safety invariants of unions

Unions currently provide *no* safety invariants of any kind. Without a documented safety invariant for a union type, code cannot make any assumptions about a union passed in from untrusted code, other than that it has a valid value, and it cannot pass a union value to untrusted code unless it could do so in purely Safe Rust.

In particular, regardless of the union's repr, it is not safe to assume that a union's field can be safely accessed, even if it seems "obviously" safe.

```rust
// Crate a
pub union U {
    pub i: i32
}
// Crate b
pub fn get_i(u: a::U) -> i32 {
    // Safe: u.i cannot be uninit in Safe Rust.
    unsafe { u.i } // UNSOUND!
}
```

Making this field access safe would require additional an safety invariant that can be understood by the compiler. The UCG WG does not oppose such a safety invariant, but believes it should be opt-in, and an RFC for such a feature is beyond our remit.

<details><summary><b>Rationale</b></summary>

At first blush, it may appear that the crate `b` is entitled to assume that it is being called from Safe Rust, or from unsafe Rust following the rules of Safe Rust. It then seems to follow that `u.i` must always be initialized, since the only safe way to create a value of type `U` is to initialize it with a value for `i`.

One might analogize this to the corresponding code with a struct:

```rust
// Crate a
pub struct S {
    pub i: i32
}
// Crate b
pub fn get_i(s: a::S) -> i32 {
    // Safe: s.i cannot be uninit in Safe Rust.
    unsafe { s.i } // Sound.
}
```

This struct code, however, is absolutely sound, even in the absence of a safety invariant documented by `S`, because of `S`'s validity invariant: for `S` to be valid, all its fields must be valid, and therefore `i` must be initialized. If it weren't, the definition of `get_i` wouldn't be the problem: the caller would be committing UB by passing an uninitialized `S`. Consequently, the `unsafe` block is redundant.

But for the union type `U`, its validity invariant is not transitive to its fields. `u.i` has no guarantee of validity for `U` to be valid.

Okay, so what about a field with a safety invariant that is stricter than the validity invariant?

```rust
// Crate a
pub struct S<'a> {
    pub s: &'a str
}
// Crate b
pub fn get_s(s: a::S<'_>) -> String {
    // Safe: s.s must be UTF-8 in Safe Rust.
    unsafe { String::from_utf8_unchecked(s.s.as_bytes().to_owned()) } // Sound.
}
```

Now we are relying on a safety invariant separate from the validity invariant: that `str` must be UTF-8. So isn't this like our union example, where we're relying on the safety invariant that `i32` can't be uninit? No, because union fields are unsafe.

Consider the following three types:

```rust
static invalid_utf8: [u8; 1] = [0xff];
pub struct Sound<'a> {
    s: &'a str
}
pub struct Unsound<'a> {
    pub s: &'a str
}
pub union AlsoSound<'a> {
    pub s: &'a str
}
impl<'a> Sound<'a> {
    pub fn new() -> Self {
        Self { s: str::from_utf8_unchecked(&invalid_utf8) }
    }
}
impl<'a> Unsound<'a> {
    pub fn new() -> Self {
        Self { s: str::from_utf8_unchecked(&invalid_utf8) }
    }
}
impl<'a> AlsoSound<'a> {
    pub fn new() -> Self {
        Self { s: str::from_utf8_unchecked(&invalid_utf8) }
    }
}
```

One struct type is `Sound`, the other is `Unsound`, and the only difference between the two is that `Unsound`'s field is `pub`. This lets us get to the heart of how safety invariants work for fields: untrusted Safe Rust cannot be allowed to get its hands on a `str` with invalid UTF-8. If it could do that, it could pass it off to arbitrary unsafe Rust that *does* assume that the `str` has UTF-8, such as the `str::chars()` method. And that will cause UB.

Thus, every struct implicitly has a safety invariant that all of its `pub` fields are safe.

The union `AlsoSound`, is identical to `Unsound` except for being a union, but it is sound. And the reason it doesn't break the rules is that union field access is unsafe. Safe Rust can call `Unsound::new().s.chars()`, but neither `Sound::new().s.chars()` nor `AlsoSound::new().s.chars()`.

It follows that unions have no safety invariants on their fields, even `pub` fields, except for those that are explicitly documented.

</details>

[Raw-repr]: ../layout/unions.md#raw-repr
[#73]: https://github.com/rust-lang/unsafe-code-guidelines/issues/73
[`MaybeUninit`]: https://doc.rust-lang.org/std/mem/union.MaybeUninit.html
[monotonicity property]: https://github.com/RalfJung/minirust/blob/master/lang/values.md#generic-properties

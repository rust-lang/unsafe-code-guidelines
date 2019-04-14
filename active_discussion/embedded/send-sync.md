# The meaning of Send and Sync in the context of interrupt handlers

The [`Send`] and [`Sync`] traits are defined in `core` as:

[`Send`]: https://doc.rust-lang.org/core/marker/trait.Send.html
[`Sync`]: https://doc.rust-lang.org/core/marker/trait.Sync.html

> `Send`: types that can be transferred across thread boundaries.

> `Sync`: types for which it is safe to share references between threads.

These definitions don't provide good guidance to embedded (microcontroller)
developers because there may or may not be (OS) threads in embedded code. Even
if you consider an interrupt handler to be semantically equivalent to a thread
these definitions would still end up being more restrictive than they need to
be.

The keyword that's missing in these definitions, in my opinion, is *preemption*.
Within an OS, preemption (context switch) could occur between any two threads
depending on how the thread scheduler is implemented. As the program can make no
assumptions about the scheduler always requiring the `Send` / `Sync` bounds is
the right call. In embedded code one can statically know whether preemption will
occur between any two interrupt handlers; if no preemption is possible then
neither of these two bounds is required for memory safety.

Note that:

- The programs in this document target the [Basic Embedded Abstract Machine
  (BEAM)][beam]. Please become familiar with the linked specification before you
  read the rest of this document.

[beam]: https://github.com/rust-lang/unsafe-code-guidelines/pull/111

- In these programs we assume that [rust-lang/rfcs#2585][rfc2585] has been
  accepted and implemented.

[rfc2585]: https://github.com/rust-lang/rfcs/pull/2585

## Cooperative interrupt handlers

Consider this program, which is more general version of [#113]

[#113]: https://github.com/rust-lang/unsafe-code-guidelines/pull/113

``` rust
#![no_std]

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("ENABLE_INTERRUPTS" : : : : "volatile");
    }

    loop {
        // .. any safe code ..
    }
}

static mut X: Type = Type::default();

#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &mut Type = unsafe { &mut X };

    // .. any safe code ..
}

#[no_mangle]
unsafe fn INTERRUPT1() {
    let x: &mut Type = unsafe { &mut X };

    // .. any safe code ..
}

#[no_mangle]
unsafe fn INTERRUPT2() {
    let x: &Type = unsafe { &X };

    // .. any safe code ..
}
```

Note that "any safe code" can *not* call `main`, `INTERRUPT0`, `INTERRUPT1` or
`INTERRUPT2` (because they are `unsafe` functions), use `asm!` or access
registers.

**Claim**: this program is well-defined / sound for any `Type`. Rust aliasing
rules are preserved because no preemption (or context switching) can occur
between `INTERRUPT0`, `INTERRUPT1` and `INTERRUPT2`.

## Preemption safe

Consider this program:

``` rust
#![no_std]

use core::{cell::UnsafeCell, ptr};

extern "C" {
    static INTERRUPT_PRIORITY: [UnsafeCell<u8>; 8];
}

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        ptr::write_volatile(INTERRUPT_PRIORITY[0].get(), 2);
        asm!("ENABLE_INTERRUPTS" : : : : "volatile");
    }

    loop {
        // .. any safe code ..
    }
}

static mut X: Type = Type::default();

// PRIORITY = 2
#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &Type = unsafe { &X };

    // .. any safe code ..
}

// PRIORITY = 1
#[no_mangle]
unsafe fn INTERRUPT1() {
    let x: &Type = unsafe { &X };

    // .. any safe code ..
}
```

**Claim**: this program is well-defined / sound if and only if `Type`
implements the `Sync` trait.

Example that shows that the bound is required: `type Type = Cell<u128>` could
result in an unsound program (data race between `main` and `INTERRUPT0`).

## Cross-priority send

Consider this program, which can be seen as a more general version of [#112].

[#112]: https://github.com/rust-lang/unsafe-code-guidelines/pull/112

``` rust
#![no_std]

use core::mem::MaybeUninit;

static mut X: MaybeUninit<Type> = MaybeUninit::uninit();

// PRIORITY = 0 (effectively)
#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        X.set(Type::default());

        asm!("ENABLE_INTERRUPTS" : : : "memory" : "volatile");
    }

    loop {
        // .. any safe code ..
    }
}

// PRIORITY = 1
#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &mut Type = unsafe { &mut *X.as_mut_ptr() };

    // .. any safe code ..
}
```

**Claim**: this program is well-defined / sound if and only if `Type`
implements the `Send` trait.

Example that shows that the bound is required: `type Type = Rc<u8>` could result
in an unsound program (data race between `main` and `INTERRUPT0`).

## Questions

- Can these programs be misoptimized by the compiler?

- Should we redefine `Send` and `Sync` (as documented in doc.rust-lang.org/std)
  in terms of preemption?

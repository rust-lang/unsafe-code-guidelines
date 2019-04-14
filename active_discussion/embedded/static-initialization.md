# Runtime initialization of static variables

Sometimes one needs to initialize a `static mut` variable, that will be
accessed by an interrupt handler, at *runtime*. For this scenario we have a
pattern where the variable starts uninitialized in const context (e.g. `static
X: _ = MaybeUninit::uninit()`) and then it's initialized *before* the first
invocation of the interrupt handler. We use a compiler fence or the "memory"
clobber to enforce strict ordering between the initialization of the static
variable and the first invocation of the interrupt handler, but is that enough?

Note that:

- The programs in this document target the [Basic Embedded Abstract Machine
  (BEAM)][beam]. Please become familiar with the linked specification before you
  read the rest of this document.

[beam]: https://github.com/rust-lang/unsafe-code-guidelines/pull/111

- In these programs we assume that [rust-lang/rfcs#2585][rfc2585] has been
  accepted and implemented.

[rfc2585]: https://github.com/rust-lang/rfcs/pull/2585

## Memory clobber synchronization

Consider this program

``` rust
#![no_std]

use core::mem::MaybeUninit;

static mut X: MaybeUninit<bool> = MaybeUninit::uninit();

#[no_mangle]
unsafe fn main() -> ! {
    X.write(false);

    unsafe {
        asm!("ENABLE_INTERRUPTS" : : : "memory" : "volatile");
        //                             ^^^^^^^^
    }

    // `INTERRUPT0` can preempt `main` from this point on and at any time

    loop {
        // .. any safe code ..
    }
}

#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &mut bool = unsafe { &mut *X.as_mut_ptr() };

    // .. any safe code ..
}
```

Note that "any safe code" can *not* call `main` or `INTERRUPT0` (because they
are `unsafe` functions), use `asm!` or access registers.

**Claim**: the memory clobber is sufficient to prevent misoptimizations.

"Why is the memory clobber required?" If the compiler reorders `X.write` to
after the `asm!` block `INTERRUPT0` could observe `X` in an uninitialized state.

## Compiler fence synchronization

Consider this program which is a slight variation of the first one:

``` rust
#![no_std]

use core::{
    cell::UnsafeCell,
    mem::MaybeUninit,
    ptr,
    sync::atomic::{self, Ordering},
};

extern "C" {
    static MASK_INTERRUPT: UnsafeCell<u8>;
    static UNMASK_INTERRUPT: UnsafeCell<u8>;
}

const ORDERING: Ordering = ..;

static mut X: MaybeUninit<bool> = MaybeUninit::uninit();

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        // mask INTERRUPT0
        ptr::write_volatile(MASK_INTERRUPT.get(), 1 << 0);
        asm!("ENABLE_INTERRUPTS" : : : : "volatile");
    }

    X.write(false);

    atomic::compiler_fence(ORDERING);

    unsafe {
        // unmask INTERRUPT0
        ptr::write_volatile(UNMASK_INTERRUPT.get(), 1 << 0);
    }

    // `INTERRUPT0` can preempt `main` from this point on and at any time

    loop {
        // .. any safe code ..
    }
}

#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &mut bool = unsafe { &mut *X.as_mut_ptr() };

    // .. any safe code ..
}
```

**Claim**: the compiler fence is sufficient to prevent misoptimization provided
that `ORDERING` is any of: `Ordering::SeqCst` `Ordering::AcqRel` or
`Ordering::Release`.

"Why is the compiler fence required?" if the compiler reorders `X.write` to
after the second `write_volatile` statement `INTERRUPT0` could observe `X` in an
uninitialized state.

## Questions

- Can these programs be misoptimized by the compiler? For example, if the
  compiler optimizes away the `X.write(..)` statement `INTERRUPT0` would observe
  an `X` in an uninitialized state. Is it allowed to do that?

  - If yes, what additional constraints are required to prevent misoptimization?
    Should `X` be initialized using a volatile write?

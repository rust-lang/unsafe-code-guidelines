# Critical sections

When two contexts (e.g. interrupt handlers) running at different priorities need
to access the same static variable some form mutual exclusion is required for
memory safety. Mutual exclusion can be implemented using a critical section on
the lower priority context where the critical section prevents the start of the
higher priority handler (preemption). We use compiler fences or the "memory"
clobber to prevent the compiler misoptimizing these critical sections but are
they enough?

Note that:

- The programs in this document target the [Basic Embedded Abstract Machine
  (BEAM)][beam]. Please become familiar with the linked specification before you
  read the rest of this document.

[beam]: https://github.com/rust-lang/unsafe-code-guidelines/pull/111

- In these programs we assume that [rust-lang/rfcs#2585][rfc2585] has been
  accepted and implemented.

[rfc2585]: https://github.com/rust-lang/rfcs/pull/2585

## Disable all interrupts (global mask)

Consider this program where a critical section is created by temporarily
disabling *all* interrupts.

``` rust
#![no_std]

static mut X: Type = Type::default();

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("ENABLE_INTERRUPTS" : : : : "volatile");
    }

    loop {
        // .. any safe code ..

        unsafe {
            // start of critical section
            asm!("DISABLE_INTERRUPTS" : : : "memory" : "volatile");
            //                              ^^^^^^^^
        }

        // `INTERRUPT0` can *not* preempt this block
        // (because all interrupts are disabled)
        {
            let x: &mut Type = unsafe {
                &mut X
            };

            // .. any safe code ..
        }

        unsafe {
            // end of critical section
            asm!("ENABLE_INTERRUPTS" : : : "memory" : "volatile");
            //                             ^^^^^^^^
        }

        // .. any safe code ..
    }
}

#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &mut Type = unsafe {
        &mut X
    };

    // .. any safe code ..
}
```

Note that "any safe code" can *not* call `main` or `INTERRUPT0` (because they
are `unsafe` functions), use `asm!` or access registers.

**Claim**:  This program is well-defined / sound if and only if `Type`
implements the `Send` trait.

Example that shows that the bound is required: `type Type = Rc<u8>` could
result in an unsound program (data race between `main` and `INTERRUPT0`).

"Why are the memory clobbers required?" Without them the compiler can reorder
`main`'s operations on `X` to outside the critical section leading to a data
race.

## Interrupt masking

Consider this program that creates a critical section by masking a single
interrupt (individual mask).

> Aside: it's also possible to implement a critical section by raising the
> running priority but the implementation of that kind of critical section is
> very similar to this one (volatile write + compiler fence) so we won't include
> it in this document.

``` rust
#![no_std]

use core::{cell::UnsafeCell, ptr, sync::atomic::{self, Ordering}};

extern "C" {
    static MASK_INTERRUPT: UnsafeCell<u8>;
    static UNMASK_INTERRUPT: UnsafeCell<u8>;
}

static mut X: Type = Type::default();

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("ENABLE_INTERRUPTS" : : : : "volatile");
    }

    loop {
        // .. any safe code ..

        unsafe {
            // start of critical section
            ptr::write_volatile(MASK_INTERRUPT.get(), 1 << 0);
        }

        atomic::compiler_fence(Ordering::SeqCst);

        // `INTERRUPT0` can *not* preempt this block
        // (because it's masked)
        {
            let x: &mut Type = unsafe {
                &mut X
            };

            // .. any safe code ..
        }

        atomic::compiler_fence(Ordering::SeqCst);

        unsafe {
            // end of critical section
            ptr::write_volatile(UNMASK_INTERRUPT.get(), 1 << 0);
        }

        // .. any safe code ..
    }
}

#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &mut Type = unsafe {
        &mut X
    };

    // .. any safe code ..
}
```

**Claim**:  This program is well-defined / sound if and only if `Type`
implements the `Send` trait.

Example that shows that the bound is required: `type Type = Rc<u8>` could
result in an unsound program (data race between `main` and `INTERRUPT0`).

"Why are the compiler fences required?" Without them the compiler can reorder
`main`'s operations on `X` to outside the critical section leading to a data
race.

## Questions

- Can these programs be misoptimized by the compiler? In particular, the
  compiler fences in the second program prevent memory operations on `X` from
  being reordered to outside the critical section but AFAIK they don't tell the
  compiler that `X` may change outside the critical section -- could the program
  cache the value of `X` on the stack? That would change the semantics of the
  program.

- I have observed that an `asm!("")` expression with *no* clobbers prevents
  operations on `static mut` variables from being merged and reordered. See
  example below. Is this intended behavior?

``` rust
#[no_mangle]
static mut X: u32 = 0;

#[no_mangle]
unsafe fn INTERRUPT0() {
    X += 1;

    asm!("");

    X += 2;
}
```

Produces this machine code (sorry for the ARM assembly)

``` armasm
INTERRUPT0:
        movw    r0, #0
        movt    r0, #8192
        ldr     r1, [r0]
        adds    r1, #1
        str     r1, [r0]  ; X += 1
        ldr     r1, [r0]
        adds    r1, #2
        str     r1, [r0]  ; X += 2
        bx      lr
```

This is the corresponding (post-optimization) LLVM IR:

``` llvm
; Function Attrs: nounwind
define void @INTERRUPT0() unnamed_addr #1 !dbg !1268 {
start:
  %0 = load i32, i32* bitcast (<{ [4 x i8] }>* @X to i32*), align 4, !dbg !1269
  %1 = add i32 %0, 1, !dbg !1269
  store i32 %1, i32* bitcast (<{ [4 x i8] }>* @X to i32*), align 4, !dbg !1269
  tail call void asm sideeffect "", ""() #5, !dbg !1270, !srcloc !1271
  %2 = load i32, i32* bitcast (<{ [4 x i8] }>* @X to i32*), align 4, !dbg !1272
  %3 = add i32 %2, 2, !dbg !1272
  store i32 %3, i32* bitcast (<{ [4 x i8] }>* @X to i32*), align 4, !dbg !1272
  ret void, !dbg !1273
}
```

## Other comments

(Feel free to disregard this section completely; it's about better optimizations
rather than misoptimizations)

`atomic::compiler_fence` and the "memory" clobber are coarse grained and they
can prevent optimization of memory accesses that don't need to be synchronized.
For example, this Rust code

``` rust
use core::{ptr, sync::atomic::{self, Ordering}};

static mut X: u32 = 0;

unsafe fn main() -> ! {
    let mut y = 0;

    // this could be part of a critical section
    atomic::compiler_fence(Ordering::SeqCst);

    y += 1;

    // prevent the compiler from optimizing away `y`
    unsafe {
       ptr::read_volatile(&y);
    }

    loop {}
}
```

produces this machine code

``` armasm
main:
        sub     sp, #4
        movs    r0, #0
        str     r0, [sp]        ; y = 0
        ldr     r0, [sp]
        adds    r0, #1
        str     r0, [sp]        ; y += 1
        ldr     r0, [sp]        ; ptr::read_volatile
        b       #-4 <main+0xe>
```

Without the compiler fence `y = 0` and `y += 1` would have been optimized into
`y = 1` resulting in shorter machine code:

``` armasm
main:
        sub     sp, #4
        movs    r0, #1
        str     r0, [sp]        ; y = 1
        ldr     r0, [sp]        ; ptr::read_volatile
        b       #-4 <main+0x8>
```

I wish we had a `atomic::compiler_fence_val(&x, ORDERING)` function that
prevented only memory operations on `x` from being reordered -- though I don't
know if LLVM supports that kind of fine grained compiler fences -- that would
result in better optimizations.

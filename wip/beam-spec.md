# The Basic Embedded Abstract Machine (BEAM)

## Summary

This document contains the specification of an *abstract machine*. The goal of
this specification is facilitating discussions about the soundness of `unsafe`
concurrent code that runs on microcontrollers.

The BEAM is meant to be representative of the microcontrollers one can run Rust
code on, as such it has concurrency capabilities commonly found in this kind of
hardware but exposes these capabilities in a way that's not tied to any
particular architecture.

## Program execution

- The BEAM has a single core / processing-unit and can only run a single process
  on this core.

- When the BEAM executes a program it uses the function called `main` as the
  entry point. Returning from this function is UNSPECIFIED behavior.

- The signature of `main` must be `[unsafe] fn() [-> !]`. That is the program
  takes no arguments.

## Interrupt handling

(Interrupt handling is a preemption mechanism that in its most basic form works
as follows: when an interrupt *signal* arrives the processor suspends the
execution of the current function and jumps to a special subroutine known as the
interrupt *handler*. When this subroutine returns the processors resumes the
execution of the function that was preempted by the interrupt handler.)

- BEAM's interrupt handling capabilities can be summarized as: prioritization of
  interrupts, individual interrupt masking and global interrupt masking.

- The BEAM has 8 *interrupts* whose *handlers* are named: `INTERRUPT0`,
  `INTERRUPT1`, all the way to `INTERRUPT7`.

- The signature of all interrupt handlers must be `[unsafe] fn()`. That is these
  handlers don't take arguments nor can they return values.

- If not defined in the program, interrupt handlers default to:

``` rust
#[no_mangle]
unsafe fn INTERRUPT_() {
    loop {}
}
```

- When the BEAM starts, all interrupts are *disabled*, *unmasked* and are
  configured with an initial *priority* of `1`; and the *running priority* of
  the system is set to `0`.

- Interrupts can be assigned any *priority* in the range `1..=255`.

- At any point in time an interrupt signal may arrive and change the state of an
  interrupt to *pending*. That is interrupt signals occur asynchronously to the
  execution of the program.

- If interrupts are *enabled*, setting the state of an *unmasked* interrupt to
  *pending* results in its handler preempting the current function *if and only
  if* the *priority* of the interrupt is *higher* than the current *running
  priority*, otherwise the interrupt will be kept in the *pending* state until
  the *running priority* is low enough to allow preemption.

- The *pending* state of an interrupt gets cleared when its handler starts
  executing.

- Once an interrupt handler starts executing the *running priority* is
  automatically set to the *priority* of the interrupt being serviced. When the
  interrupt handler returns the running priority is restored to its previous
  (before preemption) value.

- While interrupts are *disabled* preemption can not occur. Interrupts in the
  pending state will remain in that state until interrupts are re-enabled.

- Setting the state of a *masked* interrupt to *pending* will never result in
  preemption, no matter the priority of the interrupt. Such interrupt will only
  be serviced after it has been unmasked.

- The behavior of setting the *running priority* within the context of an
  interrupt handler to a value *lower* than the *priority* of that interrupt
  results in UNSPECIFIED behavior.

- Changing the *priority* of an interrupt from within its handler results in
  UNSPECIFIED behavior.

- Interrupt handling can be summarized as follows:

``` rust
// pseudo-code
if interrupts_are_enabled {
    if interrupt_N_is_pending {
        if !interrupt_N_is_masked {
            if interrupt_N_priority > running_priority {
                atomic {
                    interrupt_N_is_pending = false;
                    running_priority = interrupt_N_priority;
                    start_INTERRUPTN();
                }
            }
        }
    }
}
```

## Instructions

The BEAM has a few special instructions that can be invoked using inline
assembly `asm!`.

- `asm!("ENABLE_INTERRUPTS")`. This instruction *enables* all interrupts.

- `asm!("DISABLE_INTERRUPTS")`. This instruction *disables* all interrupts.

- `asm!("SET_RUNNING_PRIORITY $0")`. This instruction changes the *running
  priority* to the value `$0`.

## Registers

The BEAM has a special region of memory; write operations performed on this
memory region can affect how interrupt handling works. Each significant piece of
memory in this memory region is referred to as a *register*.

- All registers are byte-sized

- A program can only access the memory mapped registers by name using (part of)
  the following `extern` block.

``` rust
extern "C" {
    // write-only register
    static MASK_INTERRUPT: UnsafeCell<u8>;

    // write-only register
    static UNMASK_INTERRUPT: UnsafeCell<u8>;

    // read-write register
    static PEND_INTERRUPT: UnsafeCell<u8>;

    // read-only register
    static RUNNING_PRIORITY: UnsafeCell<u8>;

    // read-write register
    static INTERRUPT_PRIORITY: [UnsafeCell<u8>; 8];
}
```

- Performing *non*-volatile operations on these registers results in UNSPECIFIED
  behavior.

- Performing memory operations on these registers other than byte-sized reads /
  writes results in UNSPECIFIED behavior.

- The `MASK_INTERRUPT` register. Writing a value of `1` to bit `N` causes
  interrupt `N` to be *masked*. Writing a value of `0` to any bit has no effect.
  Reading this register returns an UNSPECIFIED value.

- The `UNMASK_INTERRUPT` register. Writing a value of `1` to bit `N` causes
  interrupt `N` to be *unmasked*. Writing a value of `0` to any bit has no
  effect. Reading this register returns an UNSPECIFIED value.

- The `PEND_INTERRUPT` register. Writing a value of `1` to bit `N` changes the
  state of interrupt `N` to pending. Writing a value of `0` to any bit has no
  effect. Reading this register returns the pending state of all interrupts; if
  bit `N` has value `1` it means interrupt `N` is in the pending state; if bit
  `N` has value `0` it means interrupt `N` is *not* in the pending state.

- The `RUNNING_PRIORITY` register. Reading this register returns the current
  *running priority*.

- The `INTERRUPT_PRIORITY` registers. Each of these registers is used to control
  the *priority* of the 8 interrupts the BEAM has. Writing a value `X` to the
  `i`-th register changes the static priority of interrupt `i` to `X`. Reading
  the `i`-th register returns the static priority of interrupt `i`.

## Examples

This section contains examples (real Rust code) that target the BEAM. They are
meant to improve your understanding of the above specification but they are not
part of the specification itself.

In all these examples the BEAM goes from state A, to B, to C, etc.

### Preemption

``` rust
#![no_std]

use core::{cell::UnsafeCell, ptr};

extern "C" {
    static PEND_INTERRUPT: UnsafeCell<u8>;
}

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("INTERRUPTS_ENABLE" : : : : "volatile");
    }

    // State A (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

    unsafe {
        // Set the state of `INTERRUPT0` to pending
        ptr::write_volatile(PEND_INTERRUPT.get(), 1 << 0);
    }

    // State C (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

    loop {}
}

// INTERRUPT_PRIORITY[0] = 1
#[no_mangle]
unsafe fn INTERRUPT0() {
    // State B (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)
}
```

Here setting the state of the first interrupt to *pending* causes the
`INTERRUPT0` handler to preempt the `main` function.

### Priorities

``` rust
#![no_std]

use core::{cell::UnsafeCell, ptr};

extern "C" {
    static PEND_INTERRUPT: UnsafeCell<u8>;
    static INTERRUPT_PRIORITY: [UnsafeCell<u8>; 8];
}

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        // set the priority of `INTERRUPT1` to `2`
        ptr::write_volatile(INTERRUPT_PRIORITY[1].get(), 2);
        asm!("INTERRUPTS_ENABLE" : : : : "volatile");
    }

    // State A (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

    unsafe {
        // Set the state of `INTERRUPT0` to pending
        ptr::write_volatile(PEND_INTERRUPT.get(), 1 << 0);
    }

    // State E (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

    loop {}
}

// INTERRUPT_PRIORITY[0] = 1
#[no_mangle]
unsafe fn INTERRUPT0() {
    // State B (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)

    unsafe {
        // Set the state of `INTERRUPT1` to pending
        ptr::write_volatile(PEND_INTERRUPT.get(), 1 << 1);
    }

    // State D (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)
}

// INTERRUPT_PRIORITY[1] = 2
#[no_mangle]
unsafe fn INTERRUPT1() {
    // State C (RUNNING_PRIORITY = 2, PEND_INTERRUPT = 0b0000_0000)
}
```

Higher priority interrupt handlers can preempt lower priority ones.

### Withholding an interrupt

``` rust
#![no_std]

use core::{cell::UnsafeCell, ptr};

extern "C" {
    static PEND_INTERRUPT: UnsafeCell<u8>;
    static INTERRUPT_PRIORITY: [UnsafeCell<u8>; 8];
}

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        // set the priority of `INTERRUPT0` to `2`
        ptr::write_volatile(INTERRUPT_PRIORITY[0].get(), 2);
        asm!("INTERRUPTS_ENABLE" : : : : "volatile");
    }

    // State A (RUNNING_PRIORITY = 2, PEND_INTERRUPT = 0b0000_0000)

    unsafe {
        // Set the state of INTERRUPT0 to pending
        ptr::write_volatile(PEND_INTERRUPT.get(), 1 << 0);
    }

    // State E (RUNNING_PRIORITY = 2, PEND_INTERRUPT = 0b0000_0000)

    loop {}
}

// INTERRUPT_PRIORITY[0] = 2
#[no_mangle]
unsafe fn INTERRUPT0() {
    // State B (RUNNING_PRIORITY = 2, PEND_INTERRUPT = 0b0000_0000)

    unsafe {
        // Set the state of INTERRUPT1 to pending
        ptr::write_volatile(PEND_INTERRUPT.get(), 1 << 1);
    }

    // State C (RUNNING_PRIORITY = 2, PEND_INTERRUPT = 0b0000_0010)
    //                                                          ^
}

// INTERRUPT_PRIORITY[1] = 1
#[no_mangle]
unsafe fn INTERRUPT1() {
    // State D (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)
}
```

Here `INTERRUPT1` (lower priority) is executed after `INTERRUPT0` (higher
priority) returns and before `main` (lowest priority) is resumed.

### Pending an interrupt from its handler

``` rust
#![no_std]

use core::{cell::UnsafeCell, ptr, sync::atomic::{AtomicBool, Ordering}};

extern "C" {
    static PEND_INTERRUPT: UnsafeCell<u8>;
}

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("INTERRUPTS_ENABLE" : : : : "volatile");
    }

    // State A (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

    unsafe {
        // Set the state of INTERRUPT0 to pending
        ptr::write_volatile(PEND_INTERRUPT.get(), 1 << 0);
    }

    // State E (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

    loop {}
}

// PRIORITY[0] = 1
#[no_mangle]
unsafe fn INTERRUPT0() {
    static ONCE: AtomicBool = AtomicBool::new(true);

    if ONCE.load(Ordering::Relaxed) {
        ONCE.store(false, Ordering::Relaxed);

        // State B (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)

        unsafe {
            // Set the state of INTERRUPT0 to pending
            ptr::write_volatile(PEND_INTERRUPT, 1 << 0);
        }

        // State C (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0001)
        //                                                           ^
    } else {
        // State D (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)
    }
}
```

Pending `INTERRUPT0` from its own handler has no immediate effect (because the
running priority matches the static priority of the interrupt). The second
invocation of the `INTERRUPT0` handler happens after the first invocation of
`INTERRUPT0` returns and before `main` is resumed.

### Running priority

``` rust
#![no_std]

use core::{cell::UnsafeCell, ptr};

extern "C" {
    static PEND_INTERRUPT: UnsafeCell<u8>;
}

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("INTERRUPTS_ENABLE" : : : : "volatile");

        // State A (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

        // Raise the running priority to `1`
        asm!("RUNNING_PRIORITY 1" : : :  : "volatile");

        // State B (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)

        // Set the state of INTERRUPT0 to pending
        ptr::write_volatile(PEND_INTERRUPT.get(), 1 << 0);

        // State C (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0001)
        //                                                           ^

        // Lower the running priority back to `0`
        asm!("RUNNING_PRIORITY 0" : : :  : "volatile");
    }

    // State E (RUNNING_PRIORITY = 0, PEND_INTERRUPT = 0b0000_0000)

    loop {}
}

// INTERRUPT_PRIORITY[0] = 1
#[no_mangle]
unsafe fn INTERRUPT0() {
    // State D (RUNNING_PRIORITY = 1, PEND_INTERRUPT = 0b0000_0000)
}
```

Raising the running priority defers the execution of the `INTERRUPT0` handler.

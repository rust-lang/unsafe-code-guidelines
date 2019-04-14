# Unsynchronized access to a shared `static mut` variable

Sometimes interrupt handlers need to share state. This is done using a static
variable. We claim that *no* synchronization is needed to access the shared
static variable if the interrupt handlers run at the same priority.

Note that:

- The programs in this document target the [Basic Embedded Abstract Machine
  (BEAM)][beam]. Please become familiar with the linked specification before you
  read the rest of this document.

[beam]: https://github.com/rust-lang/unsafe-code-guidelines/pull/111

- In these programs we assume that [rust-lang/rfcs#2585][rfc2585] has been
  accepted and implemented.

[rfc2585]: https://github.com/rust-lang/rfcs/pull/2585

## Example program

Consider this program

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

static mut X: u128 = 0;

#[no_mangle]
unsafe fn INTERRUPT0() {
    let x: &mut u128 = unsafe { &mut X };

    // .. any safe code ..
}

#[no_mangle]
unsafe fn INTERRUPT1() {
    let x: &mut u128 = unsafe { &mut X };

    // .. any safe code ..
}
```

Note that "any safe code" can *not* call `main`, `INTERRUPT0` or `INTERRUPT1`
(because they are `unsafe` functions), use `asm!` or access registers.

**Claim**: this program is well-defined / sound. Rust aliasing rules are
preserved because no preemption is possible between `INTERRUPT0` and
`INTERRUPT1`.

In fact, I would say that this program (ignoring `main`) is equivalent to this
code, which is sound (and reminds me of cooperative scheduling):

``` rust
let mut X: u128 = 0;

loop {
    sleep(random());

    if random() {
        INTERRUPT0(&mut X);
    } else if random() {
        INTERRUPT1(&mut X);
    }
}
```

## Questions

- Can this program be misoptimized given that the compiler has *no* information
  about `INTERRUPT0` and `INTERRUPT1` executing "cooperatively"?

# `&'static mut` references to static variables and LLVM `noalias`

Some programs need to move values between interrupt handlers (or between `main`
and an interrupt handler); this operation has to be done using static variables.
Performing this send operation "by value" can result in expensive `memcpy`s
(from handler A to the static variable and then from the variable to handler B).
A way to reduce the runtime cost is indirection: instead of memcpy-ing the value
(`T`) one can send (memcpy) a pointer that owns the value, like a `Box<T>`.

Sometimes heap allocators are avoided in embedded code for performance or
reliability reasons (e.g. a potential OOM condition is undesirable). In this
scenario `Box`, `Rc`, `Arc` and other pointers are not available to the
programmer. This leaves `&'static mut` references as the only non-`unsafe`
choice for indirection -- the `'static` lifetime is required because the
reference will be (temporarily) stored in a static variable.

We have a pattern to create a `&'static mut` reference from a `static mut`
variable, but is it sound?

Note that:

- The programs in this document target the [Basic Embedded Abstract Machine
  (BEAM)][beam]. Please become familiar with the linked specification before you
  read the rest of this document.

[beam]: https://github.com/rust-lang/unsafe-code-guidelines/pull/111

- In these programs we assume that [rust-lang/rfcs#2585][rfc2585] has been
  accepted and implemented.

[rfc2585]: https://github.com/rust-lang/rfcs/pull/2585

## Creating a `&'static mut` references

Consider this program:

``` rust
#![no_std]

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("ENABLE_INTERRUPTS" : : : : "volatile");
    }

    let x: &'static mut Type = unsafe {
        static mut X: Type = Type::new();

        &mut X
    };

    loop {
        // .. any safe code ..
    }
}

#[no_mangle]
unsafe fn INTERRUPT0() {
    // .. any safe code ..
}
```

Note that "any safe code" can *not* call `main` or `INTERRUPT0` (because they
are `unsafe` functions), use `asm!` or access registers.

**Claim**: this program is well-defined / sound for any `Type`. Rust aliasing
rules are respected because `main` is only invoked once thus `x` is the only
reference to the static variable `X`.

## Questions

@RalfJung mentioned that that the `noalias` attribute in the LLVM IR could cause
problems with this pattern because it tells LLVM that

> it (the pointer) does not alias with anything that already exists, including
> all global statics

### `noalias` argument

Would this code result in UB?

``` rust
#![no_std]

#[no_mangle]
unsafe fn main() -> ! {
    unsafe {
        asm!("ENABLE_INTERRUPTS" : : : : "volatile");
    }

    let x: &'static mut Type = unsafe {
        static mut X: Type = Type::new();

        &mut X
    };

    foo(x);

     loop {}
}

fn foo(noalias_pointer: &mut Type) {
    // .. any safe code ..
}
```

`noalias_pointer` is marked as `noalias` in the LLVM IR so it must *not* alias
with global statics like `X` but in this program `noalias_pointer` *is* a
reference to the static variable `X`.

### `noalias` return value

What about this one where `Type` is `[u8; 1024]`?

``` rust
#![no_std]

struct BumpPointerAlloc {
    ptr: *mut u8,
    space_left: usize,
}

impl BumpPointerAlloc {
    fn new(memory: &'static mut [u8]) -> Self {
        BumpPointerAlloc {
            ptr: memory.as_mut_ptr(),
            space_left: memory.len(),
        }
    }
}

unsafe impl GlobalAlloc for BumpPointerAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // returns a non-null, aligned pointer if there's enough space left
        // otherwise returns a null pointer
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // never deallocates
    }
}

unsafe fn main() -> ! {
    let x: &'static mut [u8; 1024] = unsafe {
        static mut X: [u8; 1024] = [0; 1024];

        &mut X
    };

    let alloc = BumpPointerAlloc::new(x);
    let noalias_pointer: &'static mut u32 = unsafe {
        &mut *(alloc.alloc(Layout::from_size_align(4, 4)) as *mut u32)
    };

    loop {
        // .. any safe code ..
    }
}
```

The return value of `GlobalAlloc::alloc` is marked as `noalias` meaning that
`noalias_pointer` should not overlap with a static like `X` but in this case it
does overlap with `X`.

(By the way, using a `&'static mut [u8]` that references a static variable as the
"memory pool" / "backend storage" of (global) allocators is not uncommon in
embedded programs -- there's no `brk` / `sbrk` after all.)

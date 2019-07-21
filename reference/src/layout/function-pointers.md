# Representation of Function Pointers

### Terminology

In Rust, a function pointer type, is either `fn(Args...) -> Ret`,
`extern "ABI" fn(Args...) -> Ret`, `unsafe fn(Args...) -> Ret`, or
`unsafe extern "ABI" fn(Args...) -> Ret`.
A function pointer is the address of a function,
and has function pointer type.
The pointer is implicit in the `fn` type,
and they have no lifetime of their own;
therefore, function pointers are assumed to point to
a block of code with static lifetime.
This is not necessarily always true,
since, for example, you can unload a dynamic library.
Therefore, this is _only_ a safety invariant,
not a validity invariant;
as long as one doesn't call a function pointer which points to freed memory,
it is not undefined behavior.


In C, a function pointer type is `Ret (*)(Args...)`, or `Ret ABI (*)(Args...)`,
and values of function pointer type are either a null pointer value,
or the address of a function.

### Representation

The ABI and layout of `(unsafe)? (extern "ABI")? fn(Args...) -> Ret`
is exactly that of the corresponding C type --
the lack of a null value does not change this. 
On common platforms, this means that `*const ()` and `fn(Args...) -> Ret` have
the same ABI and layout. This is, in fact, guaranteed by POSIX and Windows.
This means that for the vast majority of platforms,

```rust
fn go_through_pointer(x: fn()) -> fn() {
  let ptr = x as *const ();
  unsafe { std::mem::transmute::<*const (), fn()>(ptr) }
}
```

is both perfectly safe, and, in fact, required for some APIs -- notably,
`GetProcAddress` on Windows requires you to convert from `void (*)()` to
`void*`, to get the address of a variable;
and the opposite is true of `dlsym`, which requires you to convert from
`void*` to `void (*)()` in order to get the address of functions.
This conversion is _not_ guaranteed by Rust itself, however;
simply the implementation. If the underlying platform allows this conversion,
so will Rust.

However, null values are not supported by the Rust function pointer types --
just like references, the expectation is that you use `Option` to create
nullable pointers. `Option<fn(Args...) -> Ret>` will have the exact same ABI
as `fn(Args...) -> Ret`, but additionally allows null pointer values.


### Use

Function pointers are mostly useful for talking to C -- in Rust, you would
mostly use `T: Fn()` instead of `fn()`. If talking to a C API,
the same caveats as apply to other FFI code should be followed.
As an example, we shall implement the following C interface in Rust:

```c
struct Cons {
  int data;
  struct Cons *next;
};

struct Cons *cons(struct Cons *self, int data);

/*
  notes:
    - func must be non-null
    - thunk may be null, and shall be passed unchanged to func
    - self may be null, in which case no iteration is done
*/

void iterate(struct Cons const *self, void (*func)(int, void *), void *thunk);
bool for_all(struct Cons const *self, bool (*func)(int, void *), void *thunk);
```

```rust
# use std::{
#    ffi::c_void,
#    os::raw::c_int,
# };
#
pub struct Cons {
  data: c_int,
  next: Option<Box<Cons>>,
}

#[no_mangle]
pub extern "C" fn cons(node: Option<Box<Cons>>, data: c_int) -> Box<Cons> {
  Box::new(Cons { data, next: node })
}

#[no_mangle]
pub unsafe extern "C" fn iterate(
  node: Option<&Cons>, 
  func: unsafe extern "C" fn(i32, *mut c_void), // note - non-nullable
  thunk: *mut c_void, // note - this is a thunk, so it's just passed raw
) {
  let mut it = node;
  while let Some(node) = it {
    func(node.data, thunk);
    it = node.next.as_ref().map(|x| &**x);
  }
}

#[no_mangle]
pub unsafe extern "C" fn for_all(
  node: Option<&Cons>, 
  func: unsafe extern "C" fn(i32, *mut c_void) -> bool,
  thunk: *mut c_void,
) -> bool {
  let mut it = node;
  while let Some(node) = node {
    if !func(node.data, thunk) {
      return false;
    }
    it = node.next.as_ref().map(|x| &**x);
  }
  true
}
```

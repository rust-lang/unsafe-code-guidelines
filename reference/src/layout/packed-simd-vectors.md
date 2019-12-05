# Layout of packed SIMD vectors

**Disclaimer:** This chapter represents the consensus from issue
[#38]. The statements in here are not (yet) "guaranteed"
not to change until an RFC ratifies them.

[#38]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/38

Rust currently exposes packed[^1] SIMD vector types like `__m128` to users, but it
does not expose a way for users to construct their own vector types.

The set of currently-exposed packed SIMD vector types is
_implementation-defined_ and it is currently different for each architecture.

[^1]: _packed_ denotes that these SIMD vectors have a compile-time fixed size,
    distinguishing these from SIMD vector types whose size is only known at
    run-time. Rust currently only supports _packed_ SIMD vector types. This is
    elaborated further in [RFC2366].
    
[RFC2366]: https://github.com/gnzlbg/rfcs/blob/ppv/text/0000-ppv.md#interaction-with-cray-vectors

## Packed SIMD vector types

Packed SIMD vector types are `repr(simd)` homogeneous tuple-structs containing
`N` elements of type `T` where `N` is a power-of-two and the size and alignment
requirements of `T` are equal:

```rust,ignore
#[repr(simd)]
struct Vector<T, N>(T_0, ..., T_(N - 1));
```

The set of supported values of `T` and `N` is _implementation-defined_.

The size of `Vector` is `N * size_of::<T>()` and its alignment is an
_implementation-defined_ function of `T` and `N` greater than or equal to
`align_of::<T>()`. That is:

```rust,ignore
assert_eq!(size_of::<Vector<T, N>>(), size_of::<T>() * N);
assert!(align_of::<Vector<T, N>>() >= align_of::<T>());
```

That is, two distinct `repr(simd)` vector types that have the same `T` and the
same `N` have the same size and alignment.

Vector elements are laid out in source field order, enabling random access to
vector elements by reinterpreting the vector as an array or as an homogeneous tuple:

```rust,ignore
union U {
   vec: Vector<T, N>,
   arr: [T; N],
}

assert_eq!(size_of::<Vector<T, N>>(), size_of::<[T; N]>());
assert!(align_of::<Vector<T, N>>() >= align_of::<[T; N]>());

unsafe {
  let u = U { vec: Vector<T, N>(t_0, ..., t_(N - 1)) };

  assert_eq!(u.vec.0, u.arr[0]);
  // ... 
  assert_eq!(u.vec.(N - 1), u.arr[N - 1]);
}
```

### Unresolved questions
  
* `MaybeUninit<T>` does not have the same `repr` as `T`, so
  `MaybeUninit<Vector<T, N>>` are not `repr(simd)`, which has performance
  consequences and means that `MaybeUninit<Vector<T, N>>` is not C-FFI safe.

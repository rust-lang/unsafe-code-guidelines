# Representation of vectors

**Disclaimer:** This chapter represents the consensus from issue
[#38]. The statements in here are not (yet) "guaranteed"
not to change until an RFC ratifies them.

[#38]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/38

Rust currently exposes vector types like `__m128` to users, but it does not
expose a way for users to construct their own vector types.

The set of currently-exposed vector types is _implementation-defined_ and it is
currently different for each architecture.

## Vector types

Vector types are `repr(simd)` homogeneous tuple-structs containing `N` elements of type `T`:

```rust
#[repr(simd)]
struct Vector<T, N>(T_0, ..., T_(N - 1));
```

The set of supported values of `T` and `N` is _implementation-defined_.

The size of `Vector` is `N * size_of::<T>()` and its alignment is an
_implementation-defined_ function of `T` and `N` greater than or equal to
`align_of::<T>()`. That is:

```rust
assert_eq!(size_of::<Vector<T, N>>(), size_of::<T>() * N);
assert!(align_of::<Vector<T, N>>() >= align_of::<T>());
```

That is, two distinct `repr(simd)` vector types that have the same `T` and the
same `N` have the same size and alignment.

The layout of elements within a vector is the same as that of `[T; N]` such that
there is a a 1:1 correspondence between the indices of the vector and arrays:

```rust
union U {
   vec: Vector<T, N>,
   arr: [T; N]
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

* **Blocked**: Should the layout of vectors be the same as that of homogeneous
  tuples ? Such that:

  ```rust
  union U {
    vec: Vector<T, N>,
    tup: (T_0, ..., T_(N-1)),
  }

  assert_eq!(size_of::<Vector<T, N>>(), size_of::<(T_0, ..., T_(N-1))>());
  assert!(align_of::<Vector<T, N>>() >= align_of::<(T_0, ..., T_(N-1))>());

  unsafe {
    let u = U { vec: Vector(t_0, ..., t_(N - 1)) };

    assert_eq!(u.vec.0, u.tup.0);
    // ... 
    assert_eq!(u.vec.(N - 1), u.tup.(N - 1));
  }
  ```
  
  This is blocked on the resolution of issue [#36] about the layout of
  homogeneous structs and tuples.
  
  [#36]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/36
  
  
* Should unions containing a vector type and any number of ZSTs also be vector
  types?

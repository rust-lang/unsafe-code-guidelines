# Validity of function pointers

**Disclaimer**: This chapter is a work-in-progress. What's contained here
represents the consensus from issue [#72]. The statements in here are not (yet)
"guaranteed" not to change until an RFC ratifies them.

A function pointer is "valid" (in the sense that it can be produced without causing immediate UB) if and only if it is non-null.

That makes this code UB:

```rust
fn bad() {
    let x: fn() = unsafe { std::mem::transmute(0usize) }; // This is UB!
}
```

However, any integer value other than NULL is allowed for function pointers:

```rust
fn good() {
    let x: fn() = unsafe { std::mem::transmute(1usize) }; // This is not UB.
}
```

[#72]: https://github.com/rust-lang/unsafe-code-guidelines/issues/72

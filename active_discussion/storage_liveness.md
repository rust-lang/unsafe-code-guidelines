# Storage liveness

If you move out from a variable, can you still use the underlying stack space?

```rust
{
  let mut x: Vec<u32> = ....;
  let p: *mut Vec<u32> = &mut x;
  drop(x); // compiler can see `x` is uninitialized
        
  // what happens if you use `p` here?
} // StorageDead(x)
```

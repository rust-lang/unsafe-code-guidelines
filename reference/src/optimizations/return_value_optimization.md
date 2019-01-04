We should turn

```rust,ignore
// y unused
let mut x = f();
g(&mut x);
y = x;
// x unused
```

into

```rust,ignore
y = f();
g(&mut y);
```

to avoid a copy.

The potential issue here is `g` storing the pointer it got as an argument elsewhere.

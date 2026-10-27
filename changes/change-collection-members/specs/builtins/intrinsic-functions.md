# Intrinsic Functions — Delta: Assertion Example

> **Delta** — replaces the assertion example in the `assert` section of
> `builtins/intrinsic-functions.md`; the prose around it is unchanged.

```xulo
let xs = [2, 4, 6]
assert(xs.len == 3, "three items")  // ok: no effect
assert(xs.len == 4, "not four")     // false: stops with `panic("not four")`
```

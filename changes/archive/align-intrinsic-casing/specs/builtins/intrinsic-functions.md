# Intrinsic Functions — Delta: Intrinsic Casing

> **Delta** — edits one bullet and the `assert` example in
> `builtins/intrinsic-functions.md`.

In the `str` section, the bullet becomes:

- A base type is rendered exactly as in the table under [Output](#output); for
  a `ToString` implementor the result is the value of `toString` on the
  operand.

In the `assert` section, the example becomes:

```xulo
assert(listSize(xs) == 3, "three items")  // ok: no effect
assert(listSize(xs) == 4, "not four")     // false: stops with `panic("not four")`
```

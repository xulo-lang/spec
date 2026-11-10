# Primitive Types — Delta: Closed Range Operator

> **Delta** — replaces the `Range<T>` section in `types/primitive-types.md`.

## Range<T>

`Range<T>` is the built-in generic type produced by the range operators `..<`
(half-open) and `..=` (closed):

```xulo
let a = 0..<10     // Range<int>: excludes 10
let b = 1..=5      // Range<int>: includes 5
for i in a {
  print(i)
}
```

- The operands of a range operator MUST have the same numeric type `T`; the result type is `Range<T>`.
- A `Range<T>` is iterable: `for x in range` visits each value of type `T` from the lower bound up to — but not including, for `..<`; including, for `..=` — the upper bound.
- Ranges are values; a `Range<T>` may be bound, passed, and returned like any other value. Range patterns in `match` are specified in [`../expressions/README.md`](../expressions/README.md).

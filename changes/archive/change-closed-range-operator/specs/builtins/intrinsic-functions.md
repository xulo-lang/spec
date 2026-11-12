# Intrinsic Functions — Delta: Closed Range Operator

> **Delta** — replaces the `Range<T>` output-table row, the `asin`/`acos`
> domain sentence, and the `to_fixed` digits sentence in
> `builtins/intrinsic-functions.md`.

## Output

The `Range<T>` row of the output table becomes:

| `Range<T>` | `start..<end` or `start..=end`, per the value's own form, bounds by this table (`0..<10`, `1..=5`) |

## Math

The `asin`/`acos` bullet and the `to_fixed` bullet become:

- `Math::asin` and `Math::acos` yield `NaN` outside `-1.0..=1.0`.
- `Math::to_fixed` rounds at the `digits`-th fractional place using the same
  halfway-away-from-zero rule, keeps trailing zeros — a fixed width is the
  point of the function — and renders the sign exactly as `str` does, so a
  negative zero yields a leading `-`. The special values ignore `digits` and
  render as `NaN`, `Infinity`, and `-Infinity`. A `digits` value outside
  `0..=15` is a runtime failure
  ([`../memory-and-runtime.md`](../memory-and-runtime.md)).

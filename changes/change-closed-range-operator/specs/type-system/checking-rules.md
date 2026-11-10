# Checking Rules — Delta: Closed Range Operator

> **Delta** — replaces the range operator row and the range-pattern row in
> `type-system/checking-rules.md`.

## Expression typing

The range row of the operator typing table becomes:

| `..<` `..=` | one identical numeric type on both sides | `Range<T>` |

## Pattern typing

The range-pattern row becomes:

| `a..<b`, `a..=b` | `T` is numeric and both operands are literals of `T`'s type | none |

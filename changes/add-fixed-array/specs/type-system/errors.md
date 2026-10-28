# Errors — Delta: Fixed-Array Diagnostics

> **Delta** — adds `E0814`, `E0815`, and `E0818` to the diagnostic registry in
> `type-system/errors.md`; the `E0812`/`E0813` const-generic rows come from
> `add-const-generics`.

| `E0814` | an `Array<T, N>` is applied with a negative length | `const M = -1` then `Array<int, M>` | ``array length must be non-negative`` | [composite-types.md](../types/composite-types.md) |
| `E0815` | an array literal's element count differs from its `Array<T, N>` length | `let a: Array<int, 3> = [1, 2]` | ``array literal has 2 elements, expected 3`` | [literals.md](../expressions/literals.md) |
| `E0818` | an array literal checked against `Array<T, N>` cannot have its length compared statically — it contains a spread, or N is an abstract const parameter | `let a: Array<int, 3> = [1, ...xs]`; a literal checked against `Array<int, N>` inside `fn f<const N: int>()` | ``array literal length cannot be proven to equal N`` | [literals.md](../expressions/literals.md) |

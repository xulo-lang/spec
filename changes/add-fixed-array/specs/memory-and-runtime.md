# Memory and Runtime — Delta: Array Values

> **Delta** — assumes `change-collection-members`; adds `Array<T, N>` to the
> value-semantics table and the copy traversal, and adds the *Array index out
> of bounds* row to the closed runtime-failure table in
> `memory-and-runtime.md`; the rest is unchanged.

| **Value** | `bool`, `string`, `int`, `float`, `number`, fixed-bit numerics, `null`, `unit`, `struct`, `enum`, `Array<T, N>`, `Range<T>`, tuples | the value itself | a fresh, independent copy of `e`'s value; a `struct`, `enum`, `Array`, or tuple copy is shallow — each field or element is copied by its own category |
| **Reference** | `List<T>`, `Map<K, V>`, `Set<T>` | a reference to one shared value | the same value, shared: `x` and `e` then denote the one underlying value |

- Deep copying follows references through `List`, `Map`, and `Set` values,
  through `struct` and `enum` fields, through tuple elements, and through
  `Array` elements, and duplicates their contents.

| Condition | When it occurs | Specified in |
|-----------|----------------|--------------|
| List index or length out of range | a subscript `xs[i]` with `i` negative or at or past the list's length; a `splice` or `slice` index/count outside its stated bounds; or a negative `resize`/`truncate` length | [`types/composite-types.md`](types/composite-types.md) |
| Array index out of bounds | reading or writing `a[i]` where `i` is negative or at or past N | [`types/composite-types.md`](types/composite-types.md) |
| String index out of range | `s.codeAt(i)` with an index outside `0 … s.len − 1`, or `s.slice(start, end)` with bounds outside `0 … s.len` or with `start > end` | [`types/primitive-types.md`](types/primitive-types.md) |
| Division or remainder by zero | `/` or `%` on non-constant operands whose value is zero | [`expressions/operators.md`](expressions/operators.md) |
| Shift amount out of range | `<<` or `>>` whose amount is negative or at or past the operands' bit width, on non-constant operands | [`expressions/operators.md`](expressions/operators.md) |
| Arithmetic overflow | an `int` or fixed-bit operation whose result exceeds its type, on non-constant operands | [`types/primitive-types.md`](types/primitive-types.md) |
| `Math::toFixed` digits out of range | `Math::toFixed` is called with `digits` outside `0...=15` | [`builtins/intrinsic-functions.md`](builtins/intrinsic-functions.md) |
| Unprovided environment key | reading an `@Environment` key no provision supplies | [`components/environment.md`](components/environment.md) |
| Stack or task exhaustion | recursion depth or the runtime's task limit is exceeded | [`functions.md`](functions.md) |

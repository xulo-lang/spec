# Memory and Runtime — Delta: Runtime Failure Table

> **Delta** — replaces the closed runtime-failure table in
> `memory-and-runtime.md`; the surrounding prose is unchanged. The *Absent
> map key* row is removed, and the string row names the members.

| Condition | When it occurs | Specified in |
|-----------|----------------|--------------|
| List index or length out of range | a subscript `xs[i]` with `i` negative or at or past the list's length; a `splice` or `slice` index/count outside its stated bounds; or a negative `resize`/`truncate` length | [`types/composite-types.md`](types/composite-types.md) |
| String index out of range | `s.codeAt(i)` with an index outside `0 … s.len − 1`, or `s.slice(start, end)` with bounds outside `0 … s.len` or with `start > end` | [`types/primitive-types.md`](types/primitive-types.md) |
| Division or remainder by zero | `/` or `%` on non-constant operands whose value is zero | [`expressions/operators.md`](expressions/operators.md) |
| Shift amount out of range | `<<` or `>>` whose amount is negative or at or past the operands' bit width, on non-constant operands | [`expressions/operators.md`](expressions/operators.md) |
| Arithmetic overflow | an `int` or fixed-bit operation whose result exceeds its type, on non-constant operands | [`types/primitive-types.md`](types/primitive-types.md) |
| `Math::toFixed` digits out of range | `Math::toFixed` is called with `digits` outside `0...=15` | [`builtins/intrinsic-functions.md`](builtins/intrinsic-functions.md) |
| Unprovided environment key | reading an `@Environment` key no provision supplies | [`components/environment.md`](components/environment.md) |
| Stack or task exhaustion | recursion depth or the runtime's task limit is exceeded | [`functions.md`](functions.md) |

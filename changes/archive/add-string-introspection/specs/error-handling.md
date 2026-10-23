# Error Handling — Delta: String Index Failure

> **Delta** — adds the *String index out of range* row to the repeated
> runtime-failure table in `error-handling.md`, after *Absent map key*;
> every other row is unchanged. The table MUST stay identical to the one
> in `memory-and-runtime.md`.

the implementation, and — like the program's own `panic(...)` — it has no
handler. The table below is identical to the one in
[`memory-and-runtime.md`](memory-and-runtime.md), which is normative for the
operations themselves.

| Condition | When it occurs | Specified in |
|-----------|----------------|--------------|
| List index out of bounds | reading or writing `xs[i]` where `i` is negative or at or past the list's length | [`types/composite-types.md`](types/composite-types.md) |
| Absent map key | reading `m[k]` for a key with no binding | [`expressions/path-and-access.md`](expressions/path-and-access.md) |
| String index out of range | `stringCodeAt` with an index outside `0 … stringSize(s) − 1`, or `stringSlice` with bounds outside `0 … stringSize(s)` or with `start > end` | [`builtins/prelude.md`](builtins/prelude.md) |
| Division or remainder by zero | `/` or `%` on non-constant operands whose value is zero | [`expressions/operators.md`](expressions/operators.md) |
| Shift amount out of range | `<<` or `>>` whose amount is negative or at or past the operands' bit width, on non-constant operands | [`expressions/operators.md`](expressions/operators.md) |
| Arithmetic overflow | an `int` or fixed-bit operation whose result exceeds its type, on non-constant operands | [`types/primitive-types.md`](types/primitive-types.md) |
| `Math::toFixed` digits out of range | `Math::toFixed` is called with `digits` outside `0...=15` | [`builtins/intrinsic-functions.md`](builtins/intrinsic-functions.md) |
| Unprovided environment key | reading an `@Environment` key no provision supplies | [`components/environment.md`](components/environment.md) |
| Stack or task exhaustion | recursion depth or the runtime's task limit is exceeded | [`functions.md`](functions.md) |

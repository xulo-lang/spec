# Function Types — Delta: Const-Generic Function Values

> **Delta** — replaces the paragraph beginning “Function types may appear in
> any position” in `types/function-types.md`; the rest is unchanged.

Replace that paragraph with:

Function types may appear in any position, including as the parameter or
result of another function type, so higher-order functions are written
directly (`fn(fn(int): int): int`); the language has no separate notation for
higher-rank polymorphism. A generic function used where a function type is
expected MUST have its generic arguments determined by the expected type or
the arguments at that point; otherwise a compile-time error is reported
(`E0216`). A function with const parameters cannot be used as a bare function
value in version 1, because there is no value-position syntax for its const
arguments and they are never inferred: write a closure that supplies them,
as in `let f = fn(x: int): int { repeat<3>(x) }`. A bare use is `E0216`. The
complete compatibility and subtyping rules are in
[`type-relations.md`](type-relations.md).

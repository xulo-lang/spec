# Error Handling — Delta: Snake-Case Official Names

> **Delta** — replaces the two closed-table rows that quote `s.codeAt` and
> `Math::toFixed` in `error-handling.md`.

| String index out of range | `s.code_at(i)` with an index outside `0 … s.len − 1`, or `s.slice(start, end)` with bounds outside `0 … s.len` or with `start > end` | [`types/primitive-types.md`](types/primitive-types.md) |

| `Math::to_fixed` digits out of range | `Math::to_fixed` is called with `digits` outside `0...=15` | [`builtins/intrinsic-functions.md`](builtins/intrinsic-functions.md) |

# Errors — Delta: Const Generic Diagnostics

> **Delta** — replaces the `E0807`/`E0808` rows and adds `E0812`/`E0813` to the
> diagnostic registry in `type-system/errors.md`; the rest of the table is
> unchanged.

| `E0807` | a generic-argument list before a callee that declares no generic parameters | `f<int>(1)` where `f: fn(int): int` | `` `f` accepts no generic arguments `` | [generics.md](../types/generics.md) |
| `E0808` | a written generic-argument list of the wrong length | `first<int, 3>([1])` for `fn first<T>(…)` | ``expected 1 generic argument, found 2`` | [generics.md](../types/generics.md) |
| `E0812` | a const parameter's declared type is not `int` | `fn f<const N: string>()` | ``a const parameter must have type `int` `` | [generics.md](../types/generics.md) |
| `E0813` | a generic argument has the wrong kind — a literal or const name at a type position, or a type at a const position | `f<int>()` for `fn f<const N: int>()` | `` `int` is not a const argument `` | [generics.md](../types/generics.md) |

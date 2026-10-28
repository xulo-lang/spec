# Errors — Delta: Const Generic Diagnostics

> **Delta** — replaces the `E0112`, `E0230`, `E0807`, `E0808`, and `E0810`
> rows and adds `E0812`, `E0813`, `E0816`, and `E0817` to the diagnostic
> registry in `type-system/errors.md`; the rest of the table is unchanged.
> `E0814`, `E0815`, and `E0818` are reserved for the dependent
> `add-fixed-array` change, whose registry delta supplies those rows.

| `E0112` | `#Derive` on a type that declares generic parameters | `#Derive(ToString) struct Box<const N: int> { v: int }` | `` cannot derive `ToString` for a generic type `` | [traits.md](../types/traits.md) |
| `E0230` | a type application is ill-formed: the argument count differs from the declared generic-parameter count, a generic name is written without its arguments, a non-generic name is given arguments, or the head is a type parameter | `Map<string>` | `` `Map` expects 2 generic arguments, found 1 `` | [generics.md](../types/generics.md) |
| `E0807` | a generic-argument list before a callee that declares no generic parameters | `f<int>(1)` where `f: fn(int): int` | `` `f` accepts no generic arguments `` | [generics.md](../types/generics.md) |
| `E0808` | a written generic-argument list of the wrong length | `first<int, 3>([1])` for `fn first<T>(…)` | ``expected 1 generic argument, found 2`` | [generics.md](../types/generics.md) |
| `E0810` | a `trait` declaration declares a generic parameter list | `trait Convert<T> { fn convert(self): T }` | ``a trait may not declare generic parameters`` | [traits.md](../types/traits.md) |
| `E0812` | a const parameter's declared type is not `int` | `fn f<const N: string>()` | ``a const parameter must have type `int` `` | [generics.md](../types/generics.md) |
| `E0813` | a generic argument has the wrong kind — a const argument supplied for a type parameter, or a type argument supplied for a const parameter | `f<int>()` for `fn f<const N: int>()` | `` `int` is not a const argument `` | [generics.md](../types/generics.md) |
| `E0816` | a callee with const parameters is called without a written generic-argument list | `count()` for `fn count<const N: int>()` | `` `count` requires a written generic-argument list `` | [generics.md](../types/generics.md) |
| `E0817` | a const parameter on an `impl` does not occur in the implemented type | `impl<const N: int> Area for Circle` | ``const parameter `N` does not occur in the implemented type `` | [traits.md](../types/traits.md) |

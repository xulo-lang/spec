# Type Errors and Diagnostics — Delta: Declaration Attributes

> **Delta** — extends the name and declaration errors table of
> `type-system/errors.md` with `E0108`–`E0112`, and adds a footnote reserving
> the derive targets `Display` and `Debug`.

The name and declaration errors table, in full:

### Name and declaration errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0101` | an identifier or type name resolves to nothing; a type name where a value is required; an unknown type in a signature | `print(total)` | ``cannot find value `total` in this scope`` | [names.md](../names.md) |
| `E0102` | a member, namespace member, or enum variant does not exist | `u.namee`, `Color::Redd` | ``no member `namee` on `User` `` | [path-and-access.md](../expressions/path-and-access.md) |
| `E0103` | two declarations of one name in one scope | two `fn area` declarations | ``the name `area` is defined multiple times`` | [names.md](../names.md) |
| `E0104` | a duplicate binding in which an import participates | two imports binding `add` | `` `add` is bound twice `` | [import-export.md](../modules/import-export.md) |
| `E0105` | a type alias that expands to itself | `type A = A` | ``type alias `A` has a cyclic definition`` | [composite-types.md](../types/composite-types.md) |
| `E0106` | a module-scope declaration bears a reserved name | `fn print(…)` at file scope | ``the name `print` is reserved by the prelude`` | [prelude.md](../builtins/prelude.md) |
| `E0107` | a reference to the name of a `test` function — a call, a value use, a path, or any other mention | `helper_check()` where `test fn helper_check` exists | `` `helper_check` is a test function and cannot be referenced `` | [testing.md](../testing.md) |
| `E0108` | an attribute name that is in no registry | `#Inline` | `` attribute `Inline` is not defined `` | [traits.md](../types/traits.md) |
| `E0109` | a registered attribute on a declaration the registry does not allow it on | `#Derive(ToString) fn main()` | `` attribute `Derive` is not valid on a function `` | [traits.md](../types/traits.md) |
| `E0110` | a registered attribute whose arguments do not fit the registered shape | `#Derive` | `` attribute `Derive` takes a parenthesized list of trait names `` | [traits.md](../types/traits.md) |
| `E0111` | a well-shaped argument that names a reserved, unregistered derive target | `#Derive(Display)` | `` `Display` is not a registered derive target `` | [traits.md](../types/traits.md) |
| `E0112` | `#Derive` on a type that declares type parameters | `#Derive(ToString) struct Box<T> { v: T }` | `` cannot derive `ToString` for a generic type `` | [traits.md](../types/traits.md) |

`E0111` carries the note `` = note: registered targets: `ToString` ``.

`Display` and `Debug` are reserved names in the derive-target registry of
[`../types/traits.md`](../types/traits.md). Written as targets they are
`E0111`; nothing else in the language refers to them, and a future change
that registers them specifies what they mean.

The expansion of `#Derive` is an ordinary impl and its interpolations are
ordinary template positions, so it reuses the codes those constructs already
own: a collision with a hand-written `impl ToString` is `E0805`, and a field
or payload type with no string form is `E0801`.

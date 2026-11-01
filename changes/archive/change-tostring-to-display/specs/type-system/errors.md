# Type Errors and Diagnostics — Delta: String Form — Display, Debug, and Format

> **Delta** — updates two examples and the message of `E0112`, the trigger,
> example, and message of `E0212`, and the two footnotes after the name and
> declaration errors table of `type-system/errors.md`. No code is added,
> removed, or reassigned.

The name and declaration errors table, in full (two examples changed):

### Name and declaration errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0101` | an identifier or type name resolves to nothing; a type name where a value is required; an unknown type in a signature | `print(total)` | ``cannot find value `total` in this scope`` | [names.md](../names.md) |
| `E0102` | a field, method, or enum variant does not exist | `u.namee`, `Color::Redd` | ``no member `namee` on `User` `` | [path-and-access.md](../expressions/path-and-access.md) |
| `E0103` | two declarations of one name in one scope | two `fn area` declarations | ``the name `area` is defined multiple times`` | [names.md](../names.md) |
| `E0104` | a collision between bindings header forms create: two `use` entries, a `use` or `mod` name against a declaration, a glob overlap, a manifest path name against a local `mod` or alias | `use math::add;` then `use util::add` | `` `add` is bound twice `` | [import-export.md](../modules/import-export.md) |
| `E0105` | a type alias that expands to itself | `type A = A` | ``type alias `A` has a cyclic definition`` | [composite-types.md](../types/composite-types.md) |
| `E0106` | a module-scope declaration bears a reserved name | `fn print(…)` at file scope | ``the name `print` is reserved by the prelude`` | [prelude.md](../builtins/prelude.md) |
| `E0107` | a reference to the name of a `#Test` function — a call, a value use, a path, or any other mention | `helperCheck()` where `#Test fn helperCheck` exists | `` `helperCheck` is a test function and cannot be referenced `` | [testing.md](../testing.md) |
| `E0108` | an attribute name that is in no registry | `#Inline` | `` attribute `Inline` is not defined `` | [traits.md](../types/traits.md) |
| `E0109` | a registered attribute on a declaration the registry does not allow it on | `#Derive(Debug) fn main()` | `` attribute `Derive` is not valid on a function `` | [traits.md](../types/traits.md) |
| `E0110` | a registered attribute whose arguments do not fit the registered shape | `#Derive` | `` attribute `Derive` takes a parenthesized list of trait names `` | [traits.md](../types/traits.md) |
| `E0111` | a well-shaped argument that names a reserved, unregistered derive target | `#Derive(Display)` | `` `Display` is not a registered derive target `` | [traits.md](../types/traits.md) |
| `E0112` | `#Derive` on a type that declares generic parameters | `#Derive(Debug) struct Box<const N: int> { v: int }` | `` cannot derive `Debug` for a generic type `` | [traits.md](../types/traits.md) |
| `E0113` | a struct whose field types reach the declaring struct through fields, tuple elements, array element positions, and alias expansion with no intervening indirection | `struct Node { node: Node }` | ``type `Node` has a cyclic field definition`` | [composite-types.md](../types/composite-types.md) |

The two footnotes after that table become:

`E0111` carries the note `` = note: registered targets: `Debug` ``.

`Display` and `ToString` are reserved names in the derive-target registry of
[traits.md](../types/traits.md). Written as targets they are `E0111`;
`Debug` is the registry's one registered target, and a change that registers
a further target specifies what it means. (`Display` is a language protocol
regardless — see [traits.md](../types/traits.md) — but its form is the
author's to write and it is never derived.)

The expansion of `#Derive` is an ordinary impl, so it reuses the codes the
impl rules own: a collision with a hand-written `impl Debug` — or a second
`#Derive(Debug)` — is `E0805`, and a field or payload that is a `struct`
with no `Debug` impl is `E0801`, whose message shape
(`` `X` does not implement `Y` ``) it shares. Template interpolation, `str`,
and a rendering position that meets `unknown` — printed, expanded by a
derive, or interpolated — own `E0212` and give it the same message.

The `E0212` row of the type and expression errors table becomes:

| `E0212` | a value has no string form, in `${…}`, in `str`, at `print`, or in a `#Derive` expansion | `` `${user}` `` without `Display` | `` `User` has no string form; implement `Display` `` | [traits.md](../types/traits.md) |

`E0212` carries one of two notes, each only where its condition holds:

- raised at `print`, the note `` = note: `print` accepts a type implementing
  `Display` or `Debug` ``;
- raised anywhere for a type that implements `Debug`, the note
  `` = note: `User` implements `Debug`, which `print` accepts; string form
  requires `Display` ``.

The two never combine: a `Debug` implementor is never rejected by `print`,
so the first note implies the type has neither implementation.

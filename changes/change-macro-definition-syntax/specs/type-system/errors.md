# Type Errors and Diagnostics — Delta: Macro Definition Attributes

> **Delta** — replaces the `E1010` row and the `E1010` note sentence in
> `type-system/errors.md`. The other macro diagnostics are unchanged.

## Macro errors

Replace the `E1010` row with:

| `E1010` | a macro definition attribute is malformed (including the removed `#[macro(kind = …)]` form, a missing or extra `#[macro_derive]` target, a wrong signature, multiple definition attributes, or a declaration inside a function body), its body contains a user macro invocation, or its macro name collides with another macro or a built-in | `#[macro_derive]` with no target | ``invalid macro definition: `#[macro_derive]` requires a target`` | [type-system/checking-rules.md](checking-rules.md) |

Replace “`E1010` carries the offending key” with:

`E1010` carries the offending definition attribute or target

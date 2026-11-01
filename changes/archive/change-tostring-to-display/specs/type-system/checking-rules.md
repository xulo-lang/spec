# Checking Rules — Delta: String Form — Display, Debug, and Format

> **Delta** — renames the protocol in the template-literal row of the literal
> table and in the template requirement bullet of
> `type-system/checking-rules.md`. The requirement itself is unchanged.

The template-literal row of the literal table becomes:

| `` `…${e}…` `` | `string` | always `string`; each `e` must be a base type or `Display` (`E0212`) |

The template requirement bullet of the well-formedness list becomes:

- Every `${e}` in a template requires `e : int | float | bool | string` or a
  type implementing the built-in `Display` trait; anything else is `E0212`
  ([`../expressions/literals.md`](../expressions/literals.md)). A template
  literal always has type `string`.

# Coercion and Implicit Conversion — Delta: StringBuilder

> **Delta** — adds `StringBuilder` rows to the non-coercion table and to the
> structural equality list in `type-system/coercion.md`; the rest is
> unchanged.

Add to the “What does NOT coerce” table, after the `Map<K, V>` row:

| `StringBuilder` | `string` | `E0201` | `sb.build()` |
| `string` | `StringBuilder` | `E0201` | `StringBuilder::new()`, then `append` |

Replace the structural-equality sentence with:

Structural equality for `List`, `Array`, `Map`, `Set`, `StringBuilder`,
structs, enums, and tuples compares the contents, each pair under the same
common-type rule; for an `Array`, the length is part of the type, so arrays of
different N are unrelated and comparing them is `E0211`; for a
`StringBuilder`, the current text is compared.

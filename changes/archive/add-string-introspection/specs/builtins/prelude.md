# Prelude — Delta: String Operations

> **Delta** — replaces the reserved-names closing sentence in
> `builtins/prelude.md` and adds a new `## String operations` section
> between the List operations subsection and `## Prelude visibility`;
> everything else is unchanged.

### Reserved names

A module-scope declaration — `fn`, `struct`, `enum`, `trait`, `type`, `let`,
`let mut`, or `const` — MUST NOT bear any of the names below. Declaring one is
a compile-time error (`E0106`) reported under the diagnostic category for reserved names
in [`../type-system/errors.md`](../type-system/errors.md).

| Category | Names |
|----------|-------|
| Value and marker types | `List`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error` |
| Protocols | `ToString` |
| Namespaces | `Math`, `Time`, `Task` |
| Intrinsic functions | `print`, `println`, `str`, `assert` |

`Task` appears twice because it is both a type and a namespace. The prelude
operations defined below — collection and string alike — are reserved under
the same rule, and so is every intrinsic listed in
[`intrinsic-functions.md`](intrinsic-functions.md#complete-signature-index).

## Prelude operations

(unchanged — Map, Set, and List operations, ending with the List
operations subsection below)

### List operations

`+`, the subscript, and iteration on a list are core syntax
([`../types/composite-types.md`](../types/composite-types.md)); the prelude
supplies the operations they do not cover:

| Signature | Meaning |
|-----------|---------|
| `listSize<T>(xs: List<T>): int` | the number of elements in `xs` |
| `listPush<T>(xs: mut List<T>, v: T): unit` | appends `v` to the end of `xs` |
| `listRemove<T>(xs: mut List<T>, i: int): T?` | removes the element at index `i`, shifting later elements down, and returns it; `null` when `i` is negative or at or past the end |
| `enumerate<T>(xs: List<T>): List<(int, T)>` | one `(index, element)` tuple per element, in order |

- `listPush` and `listRemove` mutate their first argument, which MUST
  therefore be a mutable place, exactly as for `mapRemove`.
- `listRemove` is the removal that cannot fail: the subscript `xs[i]` for an
  out-of-range index is a runtime failure, while `listRemove(xs, i)` answers
  `null` and leaves `xs` unchanged.

## String operations

Strings have syntax only for concatenation (`+`), equality, and templates;
the operations that inspect them are prelude functions, reserved under the
same rule as the collection operations
([Reserved names](#reserved-names)).

| Signature | Meaning |
|-----------|---------|
| `stringSize(s: string): int` | the number of Unicode scalar values in `s` |
| `stringCodeAt(s: string, i: int): int` | the Unicode scalar value at index `i`, counting from 0 in scalar order |
| `stringSlice(s: string, start: int, end: int): string` | the substring containing the scalars at indices `start … end − 1` |
| `stringBytes(s: string): List<u8>` | the UTF-8 bytes of `s`, in order |

- Indices count Unicode scalar values: `stringSize` counts scalars, not
  bytes or UTF-16 code units, and `stringCodeAt` yields the scalar's value
  — its code point — as an `int`.
- An index outside `0 … stringSize(s) − 1`, slice bounds outside
  `0 … stringSize(s)`, or `start > end` is a runtime failure
  ([`../memory-and-runtime.md`](../memory-and-runtime.md), closed tables);
  `stringSlice(s, n, n)` is the empty string for every `n` in range, and
  `stringSlice(s, 0, stringSize(s))` is `s` itself.
- These functions are the inspection that `s[i]` refuses: the subscript on
  a `string` remains a compile-time error (`E0215`;
  [`../types/primitive-types.md`](../types/primitive-types.md)).

## Prelude visibility

(unchanged)

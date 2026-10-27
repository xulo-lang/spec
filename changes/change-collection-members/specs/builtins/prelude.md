# Prelude — Delta: Built-in Members Leave the Prelude

> **Delta** — replaces the chapter introduction, the reserved-names closing
> sentence, the `## Prelude operations` through `## String operations`
> sections (replaced by `## Built-in members`), and the Shadowing bullets of
> `builtins/prelude.md`; the built-in types, protocols, and namespaces
> sections are unchanged.

The prelude is what a module knows before it imports anything: the built-in
types, the built-in protocol, and the built-in namespaces. Nothing declares
the prelude, no import introduces it, and no import can take it away; it is
the constant background of name resolution
([`../names.md`](../names.md)). This chapter lists what the prelude contains
and the names it reserves. The built-in types' members are specified with the
types; the compiler-provided functions and the namespace members are specified
in [`intrinsic-functions.md`](intrinsic-functions.md).

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

`Task` appears twice because it is both a type and a namespace. Every
intrinsic listed in
[`intrinsic-functions.md`](intrinsic-functions.md#complete-signature-index)
is reserved under the same rule.

## Built-in members

The built-in collections and `string` expose inherent members, specified with
their types: `List`, `Map`, and `Set` in
[`../types/composite-types.md`](../types/composite-types.md), and `string` in
[`../types/primitive-types.md`](../types/primitive-types.md). They are not
prelude operations: they have no unqualified spelling, no import introduces
them, and they are not module-scope names. A local binding named `len` or
`get` does not shadow `xs.len` or `m.get`; a static member is reached through
a type name, as in `Map::fromEntries`.

## Prelude visibility

(unchanged)

## Shadowing

- **Module scope.** A module-scope declaration MUST NOT bear a reserved name
  ([Reserved names](#reserved-names)); redeclaring one is a compile-time error
  (`E0106`), not a shadowing.
- **Inner scopes.** An inner-scope `let`, `let mut`, `const`, or parameter MAY
  shadow an intrinsic name. From its declaration to the end of the enclosing
  scope the name denotes that binding alone, so a call written there reaches
  the local value rather than the intrinsic; outside the scope the intrinsic
  name is visible again. Built-in members are reached only after a receiver,
  so no local binding hides `xs.len` or `m.get`.

  ```xulo
  fn demo(): int {
    let str = "not the intrinsic"    // hides `str` for the rest of this block
    print(str)
    42
  }
  ```

- **Type names.** (unchanged)

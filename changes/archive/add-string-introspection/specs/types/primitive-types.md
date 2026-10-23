# Primitive Types — Delta: String Non-Indexability

> **Delta** — replaces the “Strings are not indexable” bullet in the string
> section of `types/primitive-types.md`; the bullets before and after it
> are unchanged.

- A string literal is written with `"` or `'`. Neither form interpolates; the backtick template form interpolates `${expr}`. Literal and template syntax is specified in [`../expressions/literals.md`](../expressions/literals.md).
- The `+` operator concatenates two strings: both operands MUST have type `string`, and the result is `string`. Mixed operands are a compile-time error (`E0227`) — `"count=" + 1` MUST NOT be written; render the value with a template — `` `count=${n}` `` — or a conversion function such as `str` (see [`../builtins/prelude.md`](../builtins/prelude.md)).
- **Strings are not indexable.** `s[i]` is NOT a defined operation for a value of type `string` and is a compile-time error (`E0215`). The language defines no indexing, slicing, or character-access syntax on `string`; inspection is done with the prelude functions `stringSize`, `stringCodeAt`, and `stringSlice`, and byte access with `stringBytes` (see [`../builtins/prelude.md`](../builtins/prelude.md)).
- `==` and `!=` are defined for two `string` operands and compare by content. The quote form is irrelevant to both type and equality: `"hello" == 'hello'` is `true`.

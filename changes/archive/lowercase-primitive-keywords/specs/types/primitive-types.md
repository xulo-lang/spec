# Primitive Types — Delta: Lowercase Primitive Keywords

> **Delta** — replaces the opening paragraph of
> `types/primitive-types.md`; the rest of the chapter is rename-through.

The intro becomes:

Primitive types are the built-in, non-composite types of Xulo. Two classes
spell differently. The **primitive value types** — the base types `bool`
and `string`, the numeric family `number`, `int`, `float`, and the
fixed-bit types `i8`…`u64`, `f32`, `f64`, and the singletons `null`,
`unit`, `unknown` — are reserved keywords: a program may use them in type
position and nowhere else, and no declaration may bear their spelling
([`../lexical-structure.md`](../lexical-structure.md)). The **marker and
generic primitives** — `View` and `Range<T>` — keep UpperCamelCase: `View`
is the return type of every component, `Range<T>` is generic, and both are
names a reader meets beside user types. No program can declare a new
primitive type of either class.

Everything after the intro — the `## bool` and `## string` sections, the
numeric tables, `null`/`unit`/`unknown`, `View`, `Range<T>` — is
rename-through: every occurrence of the old spellings in code positions
becomes the keyword spelling per the table in
[`../../proposal.md`](../../proposal.md), executed per
[`../../tasks.md`](../../tasks.md).
Section headings become the lowercase spellings (`## bool`, `## string`);
the numeric-layer headings are prose and stay.

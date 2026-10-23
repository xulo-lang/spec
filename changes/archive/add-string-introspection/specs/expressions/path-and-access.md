# Path and Access — Delta: String Intrinsics Reference

> **Delta** — replaces the “Strings are not indexable …” paragraph in the
> subscript section of `expressions/path-and-access.md`; the paragraph
> before and after it are unchanged.

The index type follows the collection: a `List<T>` is indexed by `int` and
yields `T`; a `Map<K, V>` is indexed by `K` and yields `V`. Strings are not
indexable (see [`../types/primitive-types.md`](../types/primitive-types.md));
inspect one with the prelude string functions — `stringSize`,
`stringCodeAt`, `stringSlice` — and reach its bytes with `stringBytes` (see
[`../builtins/prelude.md`](../builtins/prelude.md)).

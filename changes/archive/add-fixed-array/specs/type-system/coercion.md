# Coercion — Delta: Array Equality

> **Delta** — adds `Array` to the structural-equality sentence in
> `type-system/coercion.md`; the rest is unchanged.

Replace the structural-equality sentence with:

Structural equality for `List`, `Array`, `Map`, `Set`, structs, enums, and
tuples compares the contained values, each pair under the same common-type
rule; for an `Array`, the length is part of the type, so arrays of different N
are unrelated and comparing them is `E0211`.

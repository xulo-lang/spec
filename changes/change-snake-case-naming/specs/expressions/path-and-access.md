# Paths and Access — Delta: Snake-Case Official Names

> **Delta** — replaces the built-in-member paragraph in “Member access”, the
> path-head example in “Paths”, and the subscript paragraph in
> `expressions/path-and-access.md`.

## Member access (paragraph)

The type of a field access is the declared type of that field. On a built-in
type, the member's declared type is the type of the access: `.len`, `.size`,
and `.byte_len` are read-only properties of type `int`, and a property is not a
place — assigning to one is `E0402`. A method access has the method's function
type with `self` bound to the receiver (see
[`../functions.md`](../functions.md)); the call itself is specified in
[`calls.md`](calls.md). A name that is not in the receiver's member set is a
compile-time error (`E0102`)
(see [`../type-system/errors.md`](../type-system/errors.md)). Members are
private by default, so a member that is not `pub` is accessible only inside
the module that declares it (see [`../modules/README.md`](../modules/README.md)).

## Paths (paragraph)

A path head is a package name (through the manifest mapping), a root module
of this package, `self`, `super`, or a type name — for a variant or an
associated member, as in `Theme::Dark` and `Map::from_entries`; the head and
its resolution — including the hyphen rule that keeps `lsp-server::run()` a
path and `total - Math::PI` subtraction — are specified in
[`../modules/import-export.md`](../modules/import-export.md).

## Subscript `[]` (paragraph)

The index MUST have type `int` and the result is `T`. A subscript on a `Map`,
on a `string`, or on any other type is a compile-time error (`E0215`); a map
key is read with `m.get(k)`, and a string is inspected through its members —
`.len`, `.code_at(i)`, `.slice(start, end)`, `.byte_len`, `.bytes()`.

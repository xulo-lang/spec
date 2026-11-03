# Built-ins — Delta: Snake-Case Official Names

> **Delta** — replaces the `Map::fromEntries` reference in the “Two layers”
> section of `builtins/README.md`.

The members of `List`, `Array`, `Map`, `Set`, and `string` are part of their
types ([`../types/composite-types.md`](../types/composite-types.md),
[`../types/primitive-types.md`](../types/primitive-types.md)); they are not
prelude names and enter resolution only after a receiver or a type name
(`Map::from_entries`, `Array::filled`), so they neither reserve module-scope
names nor collide with local bindings.

# Built-ins — Delta: Array Members

> **Delta** — assumes `change-collection-members`; adds `Array` to the
> built-in-members paragraph of `builtins/README.md`; the rest is unchanged.

Replace the built-in-members paragraph with:

The members of `List`, `Array`, `Map`, `Set`, and `string` are part of their
types ([`../types/composite-types.md`](../types/composite-types.md),
[`../types/primitive-types.md`](../types/primitive-types.md)); they are not
prelude names and enter resolution only after a receiver or a type name
(`Map::fromEntries`, `Array::filled`), so they neither reserve module-scope
names nor collide with local bindings.

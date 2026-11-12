# Checking Rules — Delta: StringBuilder

> **Delta** — adds the builder to the built-in-type sentence of the member
> access rule and to the built-in static-member list in
> `type-system/checking-rules.md`; the rest is unchanged.

Replace “The built-in collection and string types declare their inherent
members with their types” with:

The built-in collection, string, and builder types declare their inherent
members with their types

Replace the static-member sentence with:

A built-in static member (`Map::from_entries`, `Set::from_list`,
`Array::filled`, `StringBuilder::new`) is reached through `::`; it has no
receiver.

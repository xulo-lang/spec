# Let and Assignment — Delta: Map Entries Through get

> **Delta** — replaces the “Map, list, and enum values are not deconstructed
> with `let`” bullet of `statements/let-and-assignment.md`; the struct and
> tuple destructuring rules are unchanged.

- **Map, list, and enum values are not deconstructed with `let`.** Map keys
  are not statically declared — read entries with `m.get(k)`
  ([`../expressions/path-and-access.md`](../expressions/path-and-access.md));
  lists and enums come apart with `match`, whose patterns are specified in
  [`../expressions/control-flow.md`](../expressions/control-flow.md).

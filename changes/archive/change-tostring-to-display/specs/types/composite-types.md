# Composite Types — Delta: String Form — Display, Debug, and Format

> **Delta** — renames the protocol in the `List.join` bullet of
> `types/composite-types.md`. The rule and its error are unchanged.

The `join` bullet becomes:

  - `.join(separator: string = ",")` — the elements' string forms joined by
    `separator`, with `""` for an empty list; type `string`. The element type
    MUST be a base type or implement `Display` (the `str` domain); otherwise
    the call is a compile-time error (`E0212`);

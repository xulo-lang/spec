# Xulo Language Specification — Delta: StringBuilder Terminology Entry

> **Delta** — adds the “string builder” term to the Terminology table of
> `README.md`; no other part of the README changes.

Add before the `struct` row:

| **string builder** | A mutable, growable UTF-8 text buffer `StringBuilder` with reference semantics; text is appended with `.append` and materialized as a `string` with `.build()`. See [`types/composite-types.md`](types/composite-types.md). |

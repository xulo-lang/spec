# Source Files — Delta: Rust-Style Attributes

> **Delta** — replaces the Test row of the declaration table and the
> `#Test` bullet in `modules/source-files.md`.

Replace the Test row:

| Form | Example | Notes |
|------|---------|-------|
| Test | `#[test] fn check() { … }` | Checked like a function body; run only by a test tool ([testing.md](../testing.md)) |

Replace the bullet with:

- A `#[test]` declaration is in neither class: its identifier binds no value —
  any reference is a compile-time error (`E0107`) — and its body follows the
  forward-reference rules of a function body
  ([testing.md](../testing.md)).

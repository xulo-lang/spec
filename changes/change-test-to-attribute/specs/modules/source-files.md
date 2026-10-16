# Source Files — Delta: `#Test` Form

> **Delta** — edits the forms table and the forward-references bullet in
> `modules/source-files.md`.

## Forms table

The Test row becomes:

| Test | `#Test fn check() { … }` | Checked like a function body; run only by a test tool ([testing.md](../testing.md)) |

Every other row is unchanged.

## Forward references

The bullet becomes:

- A `#Test` declaration is in neither class: its identifier binds no value —
  any reference is a compile-time error (`E0107`) — and its body follows the
  forward-reference rules of a function body
  ([testing.md](../testing.md)).

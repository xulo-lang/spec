# Async Expressions — Delta: Rust-Style Attributes

> **Delta** — replaces the two `#Test` mentions in
> `expressions/async-expressions.md`.

Replace the `await` legality sentence with:

- `await` is legal only inside the body of an `async` function, an `async`
  closure, or a `#[test]` function marked `async`. Elsewhere it
  is a compile-time error (`E0501`)
  ([`errors.md`](../type-system/errors.md)).

Replace the list item with:

- in a `#[test]` function that is not marked `async`;

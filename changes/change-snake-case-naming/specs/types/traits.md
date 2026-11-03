# Traits — Delta: Snake-Case Official Names

> **Delta** — replaces the member-call paragraph, the structural-versus-nominal
> `Display` bullet, and the two `toString` bullets under “Built-in formatting
> traits” in `types/traits.md`.

## Member calls (paragraph)

Beyond the two steps — when neither finds a candidate — a receiver whose
type implements `Display`, through its own type or through a `T: Display`
bound, resolves `to_string(…)` to the derived action of that impl: the
member is found without appearing in any trait's declared members,
dispatches to `display` at the default `Format`, and yields to anything
either step found (an inherent `to_string` wins at step 1, one declared by
another in-scope trait at step 2). An `impl Display` that declares
`fn to_string` remains `E0804`: the action is provided, not declared.

## Structural vs nominal (bullet)

- `Display`, `Debug`, and `Format` are built-in; no declaration introduces
  them, and the string-form rule, the derived `to_string`, and the formatting
  contracts are stated in
  [Built-in formatting traits](#built-in-formatting-traits).

## Built-in formatting traits (bullets)

- `Display` is **how a type is shown**: `display` returns the receiver's
  string form at the `Format` it is given. Every consumer that renders with
  no directive — interpolation in `` `…${…}…` ``, the intrinsic `str` (and
  a member that spells its `str` domain, `List.join`), `print`, and the
  derived `to_string` — passes the **default** `Format`,
  `Format(precision: null, alternate: false)`.
- Every `Display` impl makes the **action** available: `value.to_string()` is
  `value.display` at the default `Format`. The language provides the method
  with the impl — it dispatches through that impl, is resolved wherever the
  impl is visible (see [Member calls](#member-calls)), and is declared by
  no impl: an `impl Display` that declares
  `fn to_string` is `E0804`, because `Display` declares no such method.

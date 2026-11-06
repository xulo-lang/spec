# Intrinsic Functions — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the `#Derive(Debug)` example and the `#Test` sentence
> in `builtins/intrinsic-functions.md`.

Replace the example attribute with:

```xulo
#[derive(Debug)]
struct Point { x: int, y: int }
```

Replace the `assert` sentence with:

- `assert` is how a `#[test]` body states a requirement; the declaration that
  holds it, and what pass and fail mean, are specified in
  [`../testing.md`](../testing.md).

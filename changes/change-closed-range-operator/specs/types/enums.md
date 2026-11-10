# Enums — Delta: Closed Range Operator

> **Delta** — replaces the range-pattern bullet and its example in
> `types/enums.md`.

## Matching

The range-pattern bullet and example become:

- When the scrutinee is a number, an arm MAY use a literal or a range pattern;
  range patterns use the range operators of the expression language with their
  usual inclusivity, `0..=29` (closed) and `30..<60` (half-open):

  ```xulo
  let band = match score {
    0..=29 => "low",
    30..<60 => "medium",
    _ => "high",
  }
  ```

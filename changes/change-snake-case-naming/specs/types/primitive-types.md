# Primitive Types — Delta: Snake-Case String Members

> **Delta** — replaces the `string` inherent-member list and the following
> paragraph in `types/primitive-types.md`.

## string (inherent members)

- **Strings are not indexable.** `s[i]` is NOT a defined operation for a value
  of type `string` and is a compile-time error (`E0215`). The language defines
  no indexing syntax on `string`; a string is inspected through its inherent
  members:
  - `.len` — a read-only property of type `int`, the number of Unicode
    scalar values;
  - `.byte_len` — a read-only property of type `int`, the number of UTF-8
    bytes, O(1) and allocation-free;
  - `.code_at(i: int)` — the Unicode scalar value at index `i`, counting from
    0 in scalar order; type `int`;
  - `.slice(start: int, end: int)` — the substring containing the scalars at
    indices `start … end − 1`; type `string`;
  - `.bytes()` — a `List<u8>` holding the UTF-8 bytes, in order.
- Indices count Unicode scalar values, not bytes or UTF-16 code units;
  `code_at` yields the scalar's value — its code point — as an `int`. An index
  outside `0 … s.len − 1`, slice bounds outside `0 … s.len`, or
  `start > end` is a runtime failure
  ([`../memory-and-runtime.md`](../memory-and-runtime.md), closed tables);
  `s.slice(n, n)` is the empty string for every `n` in range, and
  `s.slice(0, s.len)` is `s` itself.

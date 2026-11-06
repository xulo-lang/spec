# Prelude — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the two `#Derive(Debug)` mentions in
> `builtins/prelude.md`.

Replace the `Debug` table row:

| Protocol | Method | Required by |
|----------|--------|-------------|
| `Debug` | `fn debug(self, format: Format): string` | the field rule of `#[derive(Debug)]`, and `print` |

Replace the `Debug` paragraph with:

- A type implements `Debug` with `impl Debug for T`, or by `#[derive(Debug)]`:
  `debug` returns the value's structural form, the counterpart of
  `Display`'s user-facing one. Nothing requires `Debug` for string form;
  the derive's field rule and `print` are its consumers.

# Macro Cache — Delta: Macro Definition Attributes

> **Delta** — replaces the macro-metadata invalidation bullet of
> `builtins/cache.md`. The key, transparency, and determinism rules are
> unchanged.

## Invalidation

Replace the macro-metadata bullet with:

- a change to the macro's definition attribute or its arguments — the
  attribute name (`#[macro]`, `#[macro_attr]`, `#[macro_derive]`), the derive
  target, or a quota key — is an implementation change;

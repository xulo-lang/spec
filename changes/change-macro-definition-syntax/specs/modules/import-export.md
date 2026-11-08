# Import and Export — Delta: Macro Definition Attributes

> **Delta** — replaces the macro row of the export mapping, the macro-name
> collision rules, and the macro rows of the well-formedness table in
> `modules/import-export.md`. The path, header, re-export, and initialization
> rules are unchanged.

## Exported names

Replace the macro row of the mapping with:

| Declaration | Exported name | What clients receive |
|-------------|---------------|----------------------|
| `pub fn f(…)` with `#[macro]`, `#[macro_attr]`, or `#[macro_derive(Target)]` | the macro name (the function identifier, or `Target`), in addition to `f` | the macro, invocable as `#[derive(Target)]`, `#[name(…)]`, or `name!(…)` |

## Name collisions

Replace the macro collision rules with:

- A `use` binding and a module-level macro definition with one macro name are
  `E0104`.
- Two macro definitions with one macro name in one module are `E1010`, not
  shadowing. For `#[macro]` and `#[macro_attr]`, a duplicate function
  identifier is the ordinary duplicate-declaration error; for
  `#[macro_derive]`, two functions with different identifiers may target the
  same name, which is `E1010`.

## Well-formedness table

Replace the two macro rows with:

| Violation | Code | Example |
|-----------|------|---------|
| Two bindings for one macro name — `use` × `use`, `use` × macro definition, glob overlap | `E0104` | `use m1::w;` then `use m2::w` |
| Two macro definitions with one macro name in one module | `E1010` | two `#[macro_derive(Serialize)]` functions |

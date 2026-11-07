# Import and Export — Delta: Macro Imports

> **Delta** — adds macro rows to the declaration-to-export mapping and the
> well-formedness table, and a “Macros” subsection to the type-only, name
> collision, and unused-name rules of `modules/import-export.md`. The path,
> header, re-export, and initialization rules are unchanged.

## Exported names

Add this row to the complete mapping:

| Declaration | Exported name | What clients receive |
|-------------|---------------|----------------------|
| `pub fn f(…)` with `#[macro(…)]` | the macro name (`target` or `name`), in addition to `f` | the macro, invocable as `#[derive(name)]`, `#[name(…)]`, or `name!(…)` |

The macro name and the function value are separate exports of one declaration:
`pub` exports both. A private `#[macro]` function exports neither.

## Type-only use and erasure

`use type` restricts value/type bindings and imports no macro. A plain `use`
imports a macro when the path names one:

```xulo
use serde::Serialize;          // imports the trait and, if exported, the derive macro
use routing::route;            // imports the attribute macro
use sql::query as q;           // aliases the function macro `query` to `q`
```

A path that resolves only to a macro is a valid `use`; it is not `E0601`. A
macro imported by `use` is used when it appears in any macro position
(`#[derive]`, `#[name]`, `name!`) or is re-exported.

## Name collisions

Macro names form a separate namespace. Add these rules:

- Two `use` bindings of one macro name are `E0104`, even if one also imports a
  value or type of that spelling.
- A `use` binding and a module-level `#[macro]` declaration with one macro name
  are `E0104`.
- A glob that contributes a macro name overlapping an existing macro binding
  is `E0104`.
- A macro name MAY equal a value or type name in the same module; the two
  namespaces do not collide.
- Two `#[macro]` declarations with one macro name in one module are `E1010`, not
  shadowing.

## Unused names

Add to the “used” definition:

- A macro binding is used when it is named in `#[derive(...)]`, in an attribute
  position (`#[name(...)]`), or as a function macro (`name!(...)`), or when it
  is re-exported by `pub use`. An unused macro import warns once, exactly as
  an unused value import does (`W0102`).

## Well-formedness table

Add these rows:

| Violation | Code | Example |
|-----------|------|---------|
| Two bindings for one macro name — `use` × `use`, `use` × declaration, glob overlap | `E0104` | `use m1::w;` then `use m2::w` |
| Two `#[macro]` declarations with one macro name in one module | `E1010` | two `#[macro(kind = "function", name = "sql")]` |

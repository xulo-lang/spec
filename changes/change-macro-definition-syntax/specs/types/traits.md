# Traits — Delta: Macro Definition Attributes

> **Delta** — replaces the built-in-attributes bullet, the built-in host table,
> and the “Macro definitions” subsection of `types/traits.md`. The invocation
> forms and the macro semantics are unchanged.

## Attributes

Replace the built-in-attributes bullet with:

- The built-in attributes are `derive`, `test`, `macro`, `macro_attr`, and
  `macro_derive`. They keep the positions and argument shapes stated below
  and in [`../grammar.md`](../grammar.md). An attribute whose name is not
  built in and not registered by an attribute macro in scope is `E0108`.

Replace the built-in host table and its note with:

  | Attribute | `fn` | `struct` | `enum` | `trait` | `impl` | `type` | `let` | `const` |
  |-----------|------|----------|--------|---------|--------|--------|-------|---------|
  | `#[derive]` | — | yes | yes | — | — | — | — | — |
  | `#[test]` | yes | — | — | — | — | — | — | — |
  | `#[macro]` | yes | — | — | — | — | — | — | — |
  | `#[macro_attr]` | yes | — | — | — | — | — | — | — |
  | `#[macro_derive]` | yes | — | — | — | — | — | — | — |
  | user attribute macro | yes | yes | yes | yes | yes | yes | yes | yes |

  `#[derive]` and `#[test]` are invocation attributes. `#[macro]`,
  `#[macro_attr]`, and `#[macro_derive]` are definition attributes: each is
  written on a module-level `fn`, and `#[macro_derive]` takes one target
  identifier. `type` is a declaration only; `let` and `const` may be
  module-level or statements. The `#[test]` and definition rows are further
  restricted by their own shape and scope rules.

## Macro definitions

Replace the “Macro definitions” subsection with:

A macro is defined by one of three built-in attributes on a module-level `fn`:

| Attribute | Macro name | Invocation | Signature |
|-----------|------------|------------|-----------|
| `#[macro]` | the function identifier | `name!(…)` | `ast::FunctionInput` → `ast::FunctionOutput` |
| `#[macro_attr]` | the function identifier | `#[name(…)]` | `ast::AttributeInput` → `ast::AttributeOutput` |
| `#[macro_derive(Target)]` | `Target` | `#[derive(Target)]` | `ast::DeriveInput` → `ast::DeriveOutput` |

```xulo
#[macro]
fn sql(input: ast::FunctionInput): Result<ast::FunctionOutput, ast::MacroFailure> { … }

#[macro_attr]
fn with_banner(input: ast::AttributeInput): Result<ast::AttributeOutput, ast::MacroFailure> { … }

#[macro_derive(Serialize)]
fn derive_serialize(input: ast::DeriveInput): Result<ast::DeriveOutput, ast::MacroFailure> { … }
```

- The macro name is the function identifier for `#[macro]` and
  `#[macro_attr]`, and the `Target` argument for `#[macro_derive]`. The
  derive implementation function's identifier is independent.
- Important: the macro's name is fully qualified by its module. `m1::w` and
  `m2::w` are different macros; a short name requires `use`.
- `pub` on the implementation function exports the macro together with the
  function value; without `pub`, both are module-private
  ([`../modules/visibility.md`](../modules/visibility.md)). For `#[macro]`
  and `#[macro_attr]` the two exports share one spelling; for
  `#[macro_derive]` the function value keeps its identifier and the macro is
  named by `Target`.
- A macro name MUST be a legal identifier and MUST NOT be a keyword or a
  reserved word. It MUST NOT be a built-in attribute name (`derive`, `test`,
  `macro`, `macro_attr`, `macro_derive`); a derive macro MUST NOT also use a
  reserved derive target (`Debug`, `Display`, `ToString`). Two macros with one
  name in one module are `E1010`; for `#[macro]` and `#[macro_attr]`, a
  duplicate function identifier is the ordinary duplicate-declaration error
  at the function level.
- `#[macro]` and `#[macro_attr]` take no positional arguments.
  `#[macro_derive]` takes exactly one positional identifier, its target. All
  three accept the optional named keys `max_steps`, `max_memory`, and
  `max_depth`, whose values are non-negative integer literals; any other
  argument is `E1010`.
- An attribute macro replaces the decorated declaration (its
  `ast::AttributeOutput.item` is required and must be a declaration of the same
  host kind) and may add items beside it. It cannot delete the declaration.
- Macros expand before global name resolution and type checking. Every macro
  implementation is checked in a definition pre-pass before it can run; it may
  not depend on an item produced by a macro in the same compilation and uses
  `ast::call_*` rather than a direct user macro call. The expanded AST is
  checked exactly like written code
  ([`../type-system/checking-rules.md`](../type-system/checking-rules.md)).
- The removed `#[macro(kind = …)]` form is a malformed `#[macro]` definition
  (`E1010`). Its note names the matching new attribute:
  `#[macro_derive(Target)]`, `#[macro_attr]`, or `#[macro]`.

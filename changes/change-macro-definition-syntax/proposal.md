# Change Macro Definition Syntax

**Status:** draft · **Date:** 2026-11-07 · **Change ID:** change-macro-definition-syntax
**Affects:** `types/traits.md`, `type-system/checking-rules.md`,
`type-system/errors.md`, `names.md`, `modules/visibility.md`,
`modules/import-export.md`, `builtins/sandbox.md`, `builtins/quota.md`,
`builtins/errors.md`, `builtins/cache.md`, `testing.md`, `expressions/calls.md`,
`ast.md`, and `introduction.md`

## Summary

Replace the single macro-definition attribute

```xulo
#[macro(kind = "derive", target = Serialize)]
fn derive_serialize(input: ast::DeriveInput): Result<…> { … }
```

with three built-in definition attributes:

```xulo
#[macro]
fn sql(input: ast::FunctionInput): Result<ast::FunctionOutput, ast::MacroFailure> { … }

#[macro_attr]
fn with_banner(input: ast::AttributeInput): Result<ast::AttributeOutput, ast::MacroFailure> { … }

#[macro_derive(Serialize)]
fn derive_serialize(input: ast::DeriveInput): Result<ast::DeriveOutput, ast::MacroFailure> { … }
```

The attribute name carries the macro kind. For `#[macro]` and `#[macro_attr]`,
the implementation function's identifier is the macro name; for
`#[macro_derive(Target)]`, `Target` is the macro name and the function
identifier is independent. The `kind`, `name`, and `target` metadata keys are
removed. Optional `max_steps`, `max_memory`, and `max_depth` keys remain, now
written as named arguments on the definition attribute.

The invocation surface is unchanged: `#[derive(Target)]`, `#[name(…)]`, and
`name!(…)` still invoke the three macro kinds.

## Motivation

The accepted macro system writes the kind twice: once as `kind = "derive"`,
`kind = "attribute"`, or `kind = "function"`, and again as the parameter type
that the kind fixes. The kind is therefore redundant with the signature. The
`name` key is redundant for attribute and function macros, whose name is
naturally the implementation function's identifier, and `target` duplicates
the derive name that `#[derive(Target)]` already writes at the invocation
site.

Three definition attributes remove both redundancies:

- the attribute name states the kind, so no `kind` string is parsed or
  diagnosed;
- `#[macro]` and `#[macro_attr]` take no macro-name argument, so the function
  identifier is the single spelling of the macro name;
- `#[macro_derive(Target)]` reads like the `#[derive(Target)]` it implements,
  and the implementation function can keep a descriptive name such as
  `derive_serialize`.

The result is shorter, closer to the way procedural macros are written in
other languages, and has fewer places where two metadata values can disagree.

## Scope

### In scope

- **Definition attributes.** `#[macro]`, `#[macro_attr]`, and
  `#[macro_derive(Target)]` on a module-level `fn`; the fixed input and output
  signatures; the optional quota keys; the definition pre-pass.
- **Names.** The function identifier is the macro name for `#[macro]` and
  `#[macro_attr]`; `Target` is the macro name for `#[macro_derive]`. The
  declaration checks, duplicate rules, visibility, and import/export rules
  follow that mapping.
- **Built-in attribute surface.** `macro_attr` and `macro_derive` join
  `derive`, `test`, and `macro` as built-in attribute names; the built-in
  host table and the reserved-name rules are updated.
- **Diagnostics.** `E1010` covers malformed definition attributes, including
  the removed `#[macro(kind = …)]` form and the wrong signature for the chosen
  attribute; `E0109` covers a definition attribute on a non-`fn` host.
- **Examples and cross-references.** Every `#[macro(kind = …)]` example and
  every generic “`#[macro]` function” reference is rewritten to the new
  definition form.

### Out of scope

- **Invocation syntax.** `#[derive(Target)]`, `#[name(…)]`, and `name!(…)` do
  not change.
- **Macro semantics.** The expansion loop, definition pre-pass, sandbox,
  quota values, context operations, expansion cache, and `ast::` records are
  unchanged except for the metadata that names the definition.
- **New macro kinds.** The change keeps exactly derive, attribute, and
  function macros.
- **Naming the implementation function independently for `#[macro]` and
  `#[macro_attr]`.** That is the cost of removing `name`; a future change
  could add an optional override if it is needed.
- **A declarative macro language.** The procedural-only rule is unchanged.

## Compatibility

- **Macro definitions are rewritten.** The old `#[macro(kind = …, …)]` form is
  removed. It is a malformed `#[macro]` definition (`E1010`) with a migration
  note that names the matching new attribute.
- **Invocation is unchanged.** A program that only invokes macros is read
  exactly as before.
- **The macro system is new.** No released program uses the old form; the
  migration affects the accepted specification and its examples, not user
  code already written against an older language version.
- **Built-in attribute names grow.** `macro_attr` and `macro_derive` join the
  built-in list, so a user macro may not take those spellings (`E1010`).
- **`add-test-options` is unaffected.** Its `#[test(...)]` grammar and
  `AttributeArg` dependency do not mention the macro definition attributes.

## Non-Goals

- **A second definition syntax.** There is one form per kind, not a `kind`
  key plus a name key.
- **Changing `#[derive]` or `#[test]`.** Their invocation forms and meanings
  are unchanged.
- **Changing the quota model.** The same three optional keys and the same
  precedence apply.

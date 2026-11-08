# Tasks — change-macro-definition-syntax

## Acceptance

- [x] Fold the deltas under [`specs/`](specs/) into `types/traits.md`,
      `type-system/checking-rules.md`, `type-system/errors.md`, `names.md`,
      `modules/visibility.md`, `modules/import-export.md`,
      `builtins/sandbox.md`, `builtins/quota.md`, `builtins/errors.md`,
      `builtins/cache.md`, `testing.md`, `expressions/calls.md`, `ast.md`, and
      `introduction.md`.
- [x] Remove every `#[macro(kind = …)]` example and every generic “`#[macro]`
      function” phrase from the main chapters.
- [x] Keep the invocation forms `#[derive(Target)]`, `#[name(…)]`, and
      `name!(…)` unchanged.
- [x] Move the directory to `archive/` and record the accepted row in
      `changes/README.md` after the fold.

## Cross-references

- [x] `types/traits.md` — the built-in attribute list names `macro_attr` and
      `macro_derive`; the host table has three `fn`-only definition rows; the
      macro-definitions subsection uses the new attributes.
- [x] `type-system/checking-rules.md` — the definition pre-pass, the signature
      table, the quota keys, the name mapping, the old-form migration, and the
      duplicate rules all use the new attributes.
- [x] `type-system/errors.md` and `builtins/errors.md` — `E1010` names the
      definition attribute and the missing/extra target; the examples use
      `#[macro_derive]`.
- [x] `names.md`, `modules/visibility.md`, and `modules/import-export.md` —
      the macro name is the function identifier for `#[macro]`/`#[macro_attr]`
      and `Target` for `#[macro_derive]`; the export and collision rules match.
- [x] `builtins/quota.md`, `builtins/cache.md`, and `builtins/sandbox.md` —
      quota keys are named arguments on the definition attribute; the cache
      invalidation key names the definition attribute; the sandbox text says
      “macro implementation function.”
- [x] `testing.md`, `expressions/calls.md`, `ast.md`, and `introduction.md` —
      the examples and generic wording use the new definition form.
- [x] `changes/add-test-options` — only the built-in attribute list changes
      in grammar/behavior; its `ast.md` delta edits only the
      `TestDeclaration` note, while this change edits only the attribute note.
      Either fold order is safe.

## Tests

- [ ] Parser tests: `#[macro]`, `#[macro_attr]`, and
      `#[macro_derive(Serialize)]` are accepted; each takes the matching
      signature; the optional quota keys are accepted on all three.
- [ ] Definition tests: `#[macro(kind = …)]` is `E1010` with a migration note;
      `#[macro_derive]` without a target, with two positional targets, or with
      a non-identifier target is `E1010`; two definition attributes on one
      function are `E1010`; a definition attribute on a non-`fn` host is
      `E0109`.
- [ ] Signature tests: `#[macro]` with `ast::AttributeInput`, `#[macro_attr]`
      with `ast::DeriveInput`, and `#[macro_derive(T)]` with
      `ast::FunctionInput` are `E1010` with a note naming the matching
      attribute.
- [ ] Name tests: two `#[macro_derive(Serialize)]` functions are `E1010`; a
      `#[macro]` function and a `#[macro_attr]` function with one identifier
      are the ordinary duplicate-declaration error; a macro name equal to
      `macro`, `macro_attr`, or `macro_derive` is `E1010`.
- [ ] Visibility tests: `pub fn sql` with `#[macro]` exports the value and the
      macro `sql`; `pub fn with_banner` with `#[macro_attr]` exports the value
      and the macro `with_banner`; `pub fn derive_serialize` with
      `#[macro_derive(Serialize)]` exports the value and the macro `Serialize`.

## Open follow-ups

- [ ] An optional name override for `#[macro]`/`#[macro_attr]` if independent
      implementation-function naming proves necessary.
- [ ] Optional richer migration tooling (an autofix or a dedicated subcode)
      for the removed `#[macro(kind = …)]` form; the basic `E1010` note is in
      scope and tested above.
- [ ] Update the external grammar snapshots only if the attribute-name
      production changes; the current grammar already admits `macro_attr` and
      `macro_derive` as identifiers.

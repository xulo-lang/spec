# Tasks — add-fixed-array

## Dependencies

- [ ] `add-const-generics` is accepted first (const parameters, explicit const
      arguments, monomorphization).
- [ ] `change-collection-members` is accepted first (List member names and
      semantics, Map/Set access).

## Deltas (written)

- [x] `specs/types/composite-types.md`: `Array<T, N>` type, members, literal,
      `Array::filled`, no dynamic members, plus the intro, List-literal, and
      field-composition wording.
- [x] `specs/expressions/literals.md`: the context-typed array literal,
      `E0815`, and the statically-unknowable-length `E0818` cases (spread,
      abstract N).
- [x] `specs/expressions/path-and-access.md`: Array subscripts.
- [x] `specs/expressions/operators.md`: `==` on Array (different lengths is
      `E0211`) and `Array` operands of `+`/arithmetic (`E0201`).
- [x] `specs/expressions/control-flow.md`: Array iteration.
- [x] `specs/type-system/checking-rules.md`: Array typing, N ≥ 0, literal
      length, the `for` iterable, and the value-composition traversal.
- [x] `specs/type-system/errors.md`: `E0113`/`E0215` rewrites and `E0814`,
      `E0815`, `E0818`.
- [x] `specs/error-handling.md` and `specs/memory-and-runtime.md`: the Array
      index row and Array value-copy semantics.
- [x] `specs/types/type-relations.md`: Array variance, subtyping, and the
      built-in-generics sentence.
- [x] `specs/builtins/prelude.md`: `Array` type and reserved name.
- [x] `specs/types/primitive-types.md`: the `Array` reserved type name.
- [x] `specs/names.md`: `Array` among the built-in non-primitive type names.
- [x] `specs/builtins/README.md` and
      `specs/builtins/intrinsic-functions.md`: Array members and rendering.
- [x] `specs/functions.md`: `Array::filled` as a built-in static member.
- [x] `specs/type-system/coercion.md` and
      `specs/components/view-syntax.md`: Array equality and component
      iteration.
- [x] `specs/types/README.md` and `specs/README.md`: type-kind, How-to-Read,
      and Terminology rows.
- [x] `specs/grammar.md`: `CallTypeArgs` recognized after a `::`-path, for
      `Array::filled<int, 3>(0)`.

## Acceptance

- [ ] Fold every file under `specs/` into the chapter its blockquote names;
      `add-const-generics` and `change-collection-members` are folded first.
- [ ] Remove the draft row from `changes/README.md`, add an `accepted` archive
      row, and move the directory to `archive/`.
- [ ] Implementation: the built-in Array type, literal construction,
      `Array::filled`, members, inline layout/copy, subscripts/iteration, the
      `E0814`/`E0815`/`E0818` diagnostics, and the Array index failure; no FFI.

## Out of scope

- [ ] Const arithmetic/inference, Array↔List conversion, Array in FFI, and
      dynamic Array members are separate changes.

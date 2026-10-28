# Tasks — add-fixed-array

## Dependencies

- [ ] `add-const-generics` is accepted first (const parameters, explicit const
      arguments, monomorphization).
- [ ] `change-collection-members` is accepted first (List member names and
      semantics, Map/Set access).

## Deltas (written)

- [x] `specs/types/composite-types.md`: `Array<T, N>` type, members, literal,
      `Array::filled`, no dynamic members.
- [x] `specs/expressions/literals.md`: the context-typed array literal,
      `E0815`, and the statically-unknowable-length `E0818` cases (spread,
      abstract N).
- [x] `specs/expressions/path-and-access.md`: Array subscripts.
- [x] `specs/expressions/operators.md`: `==` on Array (different lengths is
      `E0211`) and `Array` operands of `+`/arithmetic (`E0201`).
- [x] `specs/expressions/control-flow.md`: Array iteration.
- [x] `specs/type-system/checking-rules.md`: Array typing, N ≥ 0, literal
      length.
- [x] `specs/type-system/errors.md`: `E0814`, `E0815`, and `E0818`.
- [x] `specs/error-handling.md` and `specs/memory-and-runtime.md`: the Array
      index row and Array value-copy semantics.
- [x] `specs/types/type-relations.md`: Array variance and subtyping.
- [x] `specs/builtins/prelude.md`: `Array` type and reserved name.
- [x] `specs/types/README.md` and `specs/README.md`: type-kind and Terminology
      rows.
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

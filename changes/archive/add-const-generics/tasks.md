# Tasks — add-const-generics

## Acceptance

- [ ] Fold `specs/grammar.md`, `specs/functions.md`, `specs/names.md`,
      `specs/types/generics.md`, `specs/types/traits.md`,
      `specs/types/enums.md`, `specs/types/README.md`,
      `specs/types/function-types.md`, `specs/types/type-relations.md`,
      `specs/expressions/calls.md`, `specs/type-system/checking-rules.md`,
      `specs/type-system/errors.md`, `specs/type-system/README.md`,
      `specs/ast.md`, `specs/builtins/README.md`, and `specs/README.md` into
      the chapters their blockquotes name.
- [ ] After `grammar.md` changes, regenerate the external grammar snapshots
      (`xulo.ebnf`, `xulo.pest`, tree-sitter files) in the implementation
      repository; `machine-readable.md` itself is informative and lists those
      files rather than productions.
- [ ] Remove the draft row from `changes/README.md`, add an `accepted` archive
      row, and move the directory to `archive/`.

## Downstream

- [ ] Check `types/README.md`, `README.md`'s document map, `names.md`, and the
      notes on generic parameters for stale "type-only" wording.
- [ ] Implementation: parse `const N: int`, record the parameter kind, check
      const arguments by parameter kind at calls and type applications,
      resolve const names in the value namespace, substitute const values
      during monomorphization, match impl const parameters from the
      implemented type, and add `E0812`/`E0813`/`E0816`/`E0817`; extend
      `E0112`/`E0230`/`E0807`/`E0808`/`E0810` to generic parameters.
- [ ] Tests: a const-parameter function (`count<3>()`, `repeat<2>(5)`), a mixed
      `<T, const N: int>` function, a block-level `const` argument, a
      type/const kind mismatch, a bad const parameter type, an omitted const
      argument (`E0816`), a negative value through a `const` binding, a
      `trait` const-parameter attempt (`E0810`), an `impl` const parameter
      matched from its implemented type, an `impl` const parameter absent from
      its implemented type (`E0817`), and a const-generic function used as a
      bare value (`E0216`).
- [ ] A consumer type (fixed-length array) is a separate change, not part of
      this one.

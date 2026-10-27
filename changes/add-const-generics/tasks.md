# Tasks — add-const-generics

## Acceptance

- [ ] Fold `specs/grammar.md`, `specs/types/generics.md`,
      `specs/type-system/checking-rules.md`, `specs/type-system/errors.md`, and
      `specs/ast.md` into the chapters their blockquotes name; no other passage
      changes.
- [ ] Update `machine-readable.md` for the `GenericParam` production and the
      AST kind/constType fields.
- [ ] Remove the draft row from `changes/README.md`, add an `accepted` archive
      row, and move the directory to `archive/`.

## Downstream

- [ ] Check `types/README.md`, `README.md`'s document map, and the notes on
      generic parameters for stale "type-only" wording (none expected).
- [ ] Implementation: parse `const N: int`, record the parameter kind, check
      const arguments by parameter kind, substitute const values during
      monomorphization, and add `E0812`/`E0813`; extend `E0807`/`E0808` to
      generic arguments.
- [ ] Tests: a const-parameter function (`count<3>()`, `repeat<2>(5)`), a mixed
      `<T, const N: int>` function, a type/const kind mismatch, a bad const
      parameter type, and an omitted const argument.
- [ ] A consumer type (fixed-length array) is a separate change, not part of
      this one.

# Tasks: add-string-introspection

## Preflight

- [x] No unaccepted dependency — every delta is written against the current,
      folded chapters.

## Fold

- [x] Replace the reserved-names closing sentence and insert the
      `## String operations` section in
      [`specs/builtins/prelude.md`](specs/builtins/prelude.md).
- [x] Replace the “Strings are not indexable” bullet in
      [`specs/types/primitive-types.md`](specs/types/primitive-types.md).
- [x] Replace the subscript paragraph's string sentence in
      [`specs/expressions/path-and-access.md`](specs/expressions/path-and-access.md).
- [x] Add the *String index out of range* row in
      [`specs/memory-and-runtime.md`](specs/memory-and-runtime.md).
- [x] Add the *String index out of range* row in
      [`specs/error-handling.md`](specs/error-handling.md).

## Neighbouring edits

- [x] `type-system/errors.md` — `E0215` row unchanged (its trigger is still
      the `s[i]` subscript); no registration edit.
- [x] `expressions/literals.md` — templates and quotes untouched; no edit.
- [x] `types/composite-types.md` — `List<u8>` is ordinary element access;
      no edit.
- [x] README glossary — no new term; no edit.

## Sweep

- [x] `grep -rn 'built-in string intrinsics' --include='*.md' .` (outside
      `changes/`) — no dangling promise remains; `path-and-access.md`
      names the prelude functions.
- [x] `grep -rn 'library-level and out of scope' --include='*.md' .`
      (outside `changes/`) — the string bullet no longer says inspection is
      out of scope.
- [x] `grep -rn 'stringCode\|stringSize\|stringSlice\|stringBytes' --include='*.md' .`
      (outside `changes/`) — every occurrence is either the prelude
      definition, the two rewritten bullets, or the failure-table rows;
      nothing contradicts the scalar-index rule.

## Consistency

- [x] The two runtime-failure tables in `memory-and-runtime.md` and
      `error-handling.md` are byte-identical (row order included): 9 rows
      before, 10 after.
- [x] No new diagnostic code appears (`E0113` is not touched; `E0215` and
      `E0106` are unchanged).
- [x] The four names are reserved via the prelude rule — the
      reserved-names *table* gains no row (it lists types, protocols,
      namespaces, and intrinsics; prelude operations are covered by the
      sentence, as the collection operations already are).
- [x] `stringBytes`'s `List<u8>` result needs no new element-type rule —
      `u8` is already a fixed-bit type; no edit.
- [x] Link check across the folded files.

## Acceptance

- [x] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

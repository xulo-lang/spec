# Tasks: add-reserved-word-production

## Preflight

- [x] No unaccepted dependency — the delta is written against the current,
      folded `grammar.md`.

## Fold

- [x] Append the `ReservedWord` production to the Keywords `text` block in
      [`specs/grammar.md`](specs/grammar.md).
- [x] Add `ReservedWord` after `Keyword` in the first Production index row in
      [`specs/grammar.md`](specs/grammar.md).

## Neighbouring edits

- [x] None: `lexical-structure.md`, `names.md`, `ast.md`, and
      `type-system/errors.md` are untouched by design.

## Sweep

- [x] `grep -n 'ReservedWord' grammar.md` — four hits: the `Token`
      reference, the new production, the prose paragraph, the index row.
- [x] Every alternative of `Token` (`Identifier`, `Keyword`, `ReservedWord`,
      `Literal`, `Operator`, `Delimiter`) now has a production defined in
      `grammar.md`.

## Consistency

- [x] Definition/index bijection: every production name in the index is
      defined exactly once and every definition is indexed —
      `ReservedWord` included; the chapter's "defined exactly once" claim
      (top of chapter and index introduction) now holds.
- [x] Keyword count still 61; the reserved list in `lexical-structure.md`
      unchanged; the exclusion sentence ("The list excludes every keyword…")
      untouched.
- [x] The `? … ?` notation matches the Notation table (lexical condition).
- [x] Link check across the folded file.

## Acceptance

- [x] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

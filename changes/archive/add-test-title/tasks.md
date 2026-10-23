# Tasks: add-test-title

## Preflight

- [x] No unaccepted dependency — the deltas are written against the current,
      folded chapters (attribute registry present).

## Fold

- [x] Replace the `TestDecl` production and its two prose paragraphs in
      [`specs/grammar.md`](specs/grammar.md).
- [x] Replace the **Declaration** section and add the sentence in **The
      name** of [`specs/testing.md`](specs/testing.md).
- [x] Edit the Attributes opening paragraph and replace the `### Test` entry
      in [`specs/types/traits.md`](specs/types/traits.md).
- [x] Add the `title` field to `TestDeclaration` in
      [`specs/ast.md`](specs/ast.md).

## Neighbouring edits

- [x] `modules/source-files.md` — the Test row may keep the bare example
      (canonical form); optionally note the title in the row's third column.
      No rule change there.

## Sweep

- [x] `grep -rn 'TestDecl = ' grammar.md testing.md` — both show the group.
- [x] `grep -rn 'Test \[async\] fn' --include='*.md' .` (outside `changes/`)
      returns nothing: the old shape without the optional group is gone from
      grammar, testing, and traits.
- [x] `grep -rn 'identity as a tool reports' testing.md` — the sentence now
      says "positions or their titles".

## Consistency

- [x] `TestDecl` production identical in `grammar.md` and `testing.md`.
- [x] The `E0110` test-form message is the same string in `grammar.md` and
      `types/traits.md`; the `E0110` row in `type-system/errors.md` is
      unchanged (its trigger and example are the generic ones).
- [x] `E0107` text untouched; the testing chapter's **The name** section
      still forbids only references to the identifier.
- [x] `ast.md`: `title: StringLiteral | null` present; the no-`attributes`
      note still lists `TestDeclaration`.
- [x] Attributes opening paragraph no longer claims all arguments are trait
      names; `### Derive` still documents trait-name arguments.
- [x] Link check across the folded files.

## Acceptance

- [x] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

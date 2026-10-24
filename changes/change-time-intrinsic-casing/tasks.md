# Tasks: change-time-intrinsic-casing

## Preflight

- [ ] No unaccepted dependency — every delta is written against the current,
      folded chapters.

## Fold

- [ ] Add the **Official names** paragraph to the Naming Conventions
      preamble in [`specs/names.md`](specs/names.md).
- [ ] Replace the `Time` table, the two clock bullets, the elapsed-time
      example, and the five `Time::` index rows in
      [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md).

## Neighbouring edits

- [ ] `builtins/prelude.md` — its namespaces table links `Time` by
      heading only; no member spellings; no edit.
- [ ] `type-system/errors.md` — no diagnostic; no edit.
- [ ] `README.md` glossary — no new term; no edit.

## Sweep

- [ ] `grep -rn 'now_nanos\|monotonic_nanos' --include='*.md' .` (outside
      `changes/`) — zero hits after the fold.
- [ ] `grep -rnoE '`[A-Za-z:]*[a-z][A-Za-z0-9]*_[A-Za-z0-9_]+`' builtins/*.md`
      — zero hits: no snake_case name left in any built-in inventory.
- [ ] `grep -rn 'Official names' names.md` — the paragraph exists exactly
      once, and the RECOMMENDED sentence above it is untouched.

## Consistency

- [ ] Signature-index order unchanged: `monotonic` < `monotonicNanos` <
      `now` < `nowNanos` < `sleep`.
- [ ] The names.md paragraph lists the same official kinds the prelude
      chapter enumerates (types, protocols, namespaces, prelude
      operations, intrinsics).
- [ ] No new diagnostic code, no grammar change.
- [ ] Link check across the folded files.

## Acceptance

- [ ] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

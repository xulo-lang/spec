# Tasks: add-exit-status

## Preflight

- [x] No unaccepted dependency — every delta is written against the current,
      folded chapters.

## Fold

- [x] Insert the `## Exit status` section and replace the
      implementation-defined bullet in
      [`specs/memory-and-runtime.md`](specs/memory-and-runtime.md).
- [x] Add the exit-status bullet to the `main` list in
      [`specs/modules/source-files.md`](specs/modules/source-files.md).
- [x] Add the self-hosting note to § 1.1 of
      [`specs/README.md`](specs/README.md).

## Neighbouring edits

- [x] `error-handling.md` — its failure-table copy carries no exit-status
      claim; the panic section links to `memory-and-runtime.md`; no edit.
- [x] `ffi.md` — foreign exports keep their own return-value contract; the
      Exit status section does not mention them; no edit.
- [x] `components/view-syntax.md` — "presentation target closes" is stated
      in `memory-and-runtime.md`; no edit.

## Sweep

- [x] `grep -rn 'exit status' --include='*.md' .` (outside `changes/`) —
      every occurrence is either the new rule, its two cross-references, or
      the narrowed implementation-defined bullet; nothing still promises
      the old unqualified wording.
- [x] `grep -rn 'exits with status' --include='*.md' .` (outside
      `changes/`) — only `memory-and-runtime.md` states the rule;
      `modules/source-files.md` only cross-references it.
- [x] No `exit` intrinsic appears in `builtins/` after the fold (the FFI
      example stays in `README.md` only, clearly marked as FFI).

## Consistency

- [x] The implementation-defined list still says "four".
- [x] `README.md` § 1.1 note is informative — no `MUST` in its own right;
      it points at the normative rule in `memory-and-runtime.md`.
- [x] The `main` bullet agrees with the `## Exit status` section wording:
      "status 0" / "a nonzero status".
- [x] Link check across the folded files.

## Acceptance

- [x] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

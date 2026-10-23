# Tasks: add-wrapping-arithmetic

## Preflight

- [x] No unaccepted dependency — every delta is written against the current,
      folded chapters.

## Fold

- [x] Insert the `### Integer functions` table and append the two bullets
      in [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md).
- [x] Add the eight index rows in the complete signature index of
      [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md).
- [x] Append the cross-reference sentence to the Overflow bullet in
      [`specs/types/primitive-types.md`](specs/types/primitive-types.md).

## Neighbouring edits

- [x] `builtins/prelude.md` — `Math` is listed as a namespace with no
      member inventory; the reserved-names table is unchanged; no edit.
- [x] `memory-and-runtime.md` / `error-handling.md` — no new runtime
      failure (the wrapping functions are total, the rotates accept every
      amount); the tables stay at their current row count; no edit.
- [x] `type-system/errors.md` — `E0210` trigger unchanged; no edit.

## Sweep

- [x] `grep -rn 'wrapping' --include='*.md' .` (outside `changes/`) —
      every occurrence is the new `Math` members, the two bullets, or the
      Overflow cross-reference; nothing claims operators wrap.
- [x] `grep -rn 'popCount\|countLeadingZeros\|countTrailingZeros\|rotateLeft\|rotateRight' --include='*.md' .`
      (outside `changes/`) — table, bullets, and index only.
- [x] No `Math::` member name collides with an existing index row or a
      prelude function name.

## Consistency

- [x] The complete signature index stays alphabetical under the existing
      case-insensitive order (`PI` < `popCount` < `pow`;
      `cos` < `countLeadingZeros`; `pow` < `rotateLeft` < `round`;
      `wrapping*` after `trunc`).
- [x] The Overflow bullet's appended sentence does not soften the trap
      rule — it scopes the exception to the named intrinsics only.
- [x] No new diagnostic code, no grammar change, no new runtime-failure
      row anywhere.
- [x] Link check across the folded files.

## Acceptance

- [x] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

# Tasks: change-shift-count

## Preflight

- [x] No unaccepted dependency — every delta is written against the current,
      folded chapters.

## Fold

- [x] Replace the shift paragraph in
      [`specs/expressions/operators.md`](specs/expressions/operators.md).
- [x] Replace the `E0210` row in
      [`specs/type-system/errors.md`](specs/type-system/errors.md).
- [x] Replace the shift row and the constant-evaluation sentence in
      [`specs/type-system/checking-rules.md`](specs/type-system/checking-rules.md).
- [x] Replace the constant-arithmetic sentence in
      [`specs/type-system/coercion.md`](specs/type-system/coercion.md).
- [x] Replace the example block and the constant-evaluation bullet in
      [`specs/statements/let-and-assignment.md`](specs/statements/let-and-assignment.md).
- [x] Replace the runtime-failure table in
      [`specs/memory-and-runtime.md`](specs/memory-and-runtime.md).
- [x] Replace the runtime-failure table in
      [`specs/error-handling.md`](specs/error-handling.md).

## Neighbouring edits

- [x] `types/primitive-types.md` — the Overflow bullet stays overflow-only
      (div-by-zero is not there either); no rule change.
- [x] `README.md` glossary "runtime failure" example list stays
      non-exhaustive; no edit.

## Sweep

- [x] `grep -rn 'MUST be a non-negative integer' --include='*.md' .` (outside
      `changes/`) — only `expressions/operators.md` carries the full domain
      rule; `checking-rules.md` carries the shortened row form.
- [x] `grep -rn 'shifts by a negative or out-of-range' --include='*.md' .`
      (outside `changes/`) — hits `operators.md` (or `E0210` row),
      `errors.md`, and nothing contradicts it.
- [x] No occurrence of the old trigger ``overflows its type | `const H```
      without the shift clause remains in `type-system/errors.md`.

## Consistency

- [x] The two runtime-failure tables in `memory-and-runtime.md` and
      `error-handling.md` are byte-identical (row order included).
- [x] `E0210` stays a single registered code — the registered-code count is
      unchanged; no new `E`/`W` number appears anywhere.
- [x] Constant-condition enumerations all four places agree: `operators.md`
      (shift paragraph), `checking-rules.md` (two places), `coercion.md`,
      `let-and-assignment.md`.
- [x] Width wording identical everywhere it appears: "64 for `int`, `W` for
      `usize` and `isize`".
- [x] Link check across the folded files.

## Acceptance

- [x] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

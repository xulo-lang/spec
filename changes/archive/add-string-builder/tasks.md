# Tasks: add-string-builder

## Preflight

- [x] No unaccepted dependency — every delta is written against the current,
      folded chapters.
- [x] No grammar, AST, diagnostic, or runtime-failure delta is in scope; the
      type is reached through existing path and member syntax.

## Fold

- [x] Replace the chapter introduction and insert `## StringBuilder` after
      `## Set<T>` in
      [`specs/types/composite-types.md`](specs/types/composite-types.md).
- [x] Add the Builder kind row and update the composite-types bullet in
      [`specs/types/README.md`](specs/types/README.md).
- [x] Add `StringBuilder` to the reserved-name list and the composite-capable
      sentence in
      [`specs/types/primitive-types.md`](specs/types/primitive-types.md).
- [x] Add `StringBuilder` to the built-in types table, the reserved names
      table, and the built-in-members paragraph in
      [`specs/builtins/prelude.md`](specs/builtins/prelude.md).
- [x] Add the output rendering row and the `str` rejection mention in
      [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md).
- [x] Add `StringBuilder` to the built-in non-primitive type names in
      [`specs/names.md`](specs/names.md).
- [x] Add `StringBuilder` to the reference row, the equality paragraph, and
      the first `copy` bullet in
      [`specs/memory-and-runtime.md`](specs/memory-and-runtime.md).
- [x] Add `StringBuilder` to the structural-equality paragraph in
      [`specs/expressions/operators.md`](specs/expressions/operators.md).
- [x] Update the built-in-type and static-member sentences in
      [`specs/type-system/checking-rules.md`](specs/type-system/checking-rules.md).
- [x] Add the two non-coercion rows and the equality mention in
      [`specs/type-system/coercion.md`](specs/type-system/coercion.md).
- [x] Add the “string builder” term to the Terminology table in
      [`specs/README.md`](specs/README.md).

## Neighbouring edits

- [x] `type-system/errors.md` — no new code; `E0106`, `E0102`, `E0211`,
      `E0212`, `E0213`, `E0215`, and `E0404` already cover every rejection;
      no registration edit.
- [x] `functions.md` and `expressions/path-and-access.md` — the built-in
      inherent-member sentences gain the builder.
- [x] `expressions/control-flow.md` — the iterable list is unchanged; a
      builder is not iterable (`E0213`).
- [x] `expressions/path-and-access.md` — static members are already path calls;
      no edit.
- [x] `types/type-relations.md` — `StringBuilder` is non-generic and nominal;
      no variance row and no subtyping rule are added.
- [x] `builtins/prelude.md` — the protocols table is unchanged; the builder
      has no `Display` implementation.
- [x] `builtins/README.md` — the member-bearing type list gains
      `StringBuilder`, and `StringBuilder::new` joins the static examples.
- [x] `types/enums.md` — enum payload equality names `StringBuilder` among
      the content-bearing slot types.
- [x] `ffi.md` — `StringBuilder` joins the non-ABI type examples.

## Sweep

- [x] `grep -rn 'List.*,.*Map.*,.*Set' --include='*.md' .` (outside
      `changes/`) — every built-in type enumeration that also names
      `StringBuilder`'s peers either gains it or is intentionally about
      collections only.
- [x] `grep -rn 'StringBuilder' --include='*.md' .` (outside `changes/`) —
      every occurrence is the type section, a prelude/name reservation, a
      memory/equality/coercion mention, the rendering row, or the glossary
      term; nothing contradicts the reference or no-`Display` rules.
- [x] `grep -rn 'string builder' --include='*.md' .` (outside `changes/`) —
      no stale term definition is missing.

## Consistency

- [x] `StringBuilder` is reserved in `builtins/prelude.md`,
      `types/primitive-types.md`, and `names.md`.
- [x] The reference category, `copy`, and structural equality agree across
      `memory-and-runtime.md`, `expressions/operators.md`, and
      `type-system/coercion.md`.
- [x] `print(sb)` renders the buffer text, while `str(sb)` and `${sb}` remain
      `E0212`; `.build()` is named as the materialization in both places.
- [x] No new diagnostic code, no new runtime-failure row, and no grammar or
      AST change appears.
- [x] Link check across the folded files.

## Acceptance

- [x] All boxes above checked; move the directory to `archive/` as `accepted`;
      move the index entry in [`../README.md`](../README.md).

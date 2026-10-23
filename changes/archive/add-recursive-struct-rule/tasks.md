# Tasks: add-recursive-struct-rule

## Preflight

- [x] No unaccepted dependency — every delta is written against the current,
      folded chapters.

## Fold

- [x] Add the composition-cycle bullet to the struct bullet list in
      [`specs/types/composite-types.md`](specs/types/composite-types.md),
      after the nominality bullet.
- [x] Add the payload-recursion bullet to
      [`specs/types/enums.md`](specs/types/enums.md), after the
      “payload slot may have any type” bullet.
- [x] Add the `E0113` row after `E0112` in
      [`specs/type-system/errors.md`](specs/type-system/errors.md).
- [x] Add the **Value composition** bullet after **Type aliases** in
      [`specs/type-system/checking-rules.md`](specs/type-system/checking-rules.md).
- [x] Replace the Declarations row in
      [`specs/type-system/README.md`](specs/type-system/README.md).

## Neighbouring edits

- [x] `types/generics.md` — the “describes finite values and needs no
      special rule” sentence is compatible (a `List<Node<T>>` field is an
      indirection); no edit.
- [x] `type-system/errors.md` range table — `E0101`–`E0199` already covers
      `E0113`; no range edit.
- [x] README glossary — no new term; no edit.

## Sweep

- [x] `grep -rn 'E0113' --include='*.md' .` (outside `changes/`) — hits
      `type-system/errors.md`, `types/composite-types.md`,
      `type-system/checking-rules.md`; nothing else needs the code.
- [x] `grep -rn 'E0105' --include='*.md' .` (outside `changes/`) — every
      occurrence is alias-specific; no struct case smuggled in.
- [x] `grep -rn 'cyclic' --include='*.md' .` (outside `changes/`) — the two
      cycle diagnostics (`E0105` alias, `E0113` struct) do not share a
      message.

## Consistency

- [x] Registered-code count rises by exactly one (`E0113`); the next free
      code is `E0114`.
- [x] The indirection list — `List<T>`, `Map<K, V>`, `Set<T>`, `T?`, a
      function type, or an enum type — is worded identically in
      `composite-types.md` and `checking-rules.md`.
- [x] `types/enums.md` states the permission without re-listing the
      indirections (structs own the list; enums reference it).
- [x] Link check across the folded files.

## Acceptance

- [x] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).

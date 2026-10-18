# Tasks: align-intrinsic-casing

## Preflight

- [x] No unaccepted dependency — the deltas are written against the current,
      folded chapters.

## Fold

- [ ] Replace the `ToString` row and bullet, the example block, and the Map /
      Set / List tables and bullets in
      [`specs/builtins/prelude.md`](specs/builtins/prelude.md).
- [ ] Edit the `str` bullet in
      [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md).
- [ ] Replace the `ToString` example and the `#Derive` bullet in
      [`specs/types/traits.md`](specs/types/traits.md).
- [ ] Edit the Declaration example line in
      [`specs/testing.md`](specs/testing.md).

## Neighbouring edits

- [ ] `changes/add-test-title/` — if that change is still unaccepted, update
      its `specs/testing.md` delta example to `mapGet` and its `proposal.md`
      example if it mentions `map_get`; if it already folded, no edit — the
      sweep below is the check either way.

## Sweep

- [ ] `grep -rn 'list_push\|list_remove\|list_size\|map_from_entries\|map_get\|map_has_key\|map_remove\|map_size\|map_to_entries\|set_contains\|set_from_list\|set_insert\|set_remove\|set_size\|to_string' --include='*.md' .` (outside `changes/archive/` and this change's own files) returns nothing.
- [ ] `grep -rn '`[a-z][a-z0-9]*_[a-z_0-9]*' builtins/ types/traits.md testing.md` outside archive returns no public-name match (only prose snake_case like variable comments would remain — verify none).

## Consistency

- [ ] The four prelude sections and the example block all use the new
      spellings; `enumerate` and the single-word intrinsics are untouched.
- [ ] `ToString` method spelled `toString` in the protocol table, the traits
      chapter, and the `intrinsic-functions.md` bullet — three places, one
      string.
- [ ] `type-system/errors.md` unchanged: no message or trigger quotes an
      intrinsic name; `E0106` still uses `print`.
- [ ] `names.md` unchanged: the convention table already says lowerCamelCase
      for functions and methods.
- [ ] Link check across the folded files.

## Acceptance

- [ ] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../README.md).

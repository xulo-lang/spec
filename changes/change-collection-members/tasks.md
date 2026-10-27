# Tasks — change-collection-members

## Acceptance

- [ ] Fold each file under `specs/` into the chapter its blockquote names; no
      other passage changes.
- [ ] Remove the draft row from `changes/README.md`; add an `accepted` row to
      the archive index; move the directory to `archive/`.
- [ ] Outside `changes/archive/`, confirm none of these remain:
      `mapSize`, `mapGet`, `mapHasKey`, `mapRemove`, `mapToEntries`,
      `mapFromEntries`,
      `listSize`, `listPush`, `listRemove`, `enumerate`, `setFromList`,
      `setInsert`, `setRemove`, `setContains`, `setSize`, `stringSize`,
      `stringCodeAt`, `stringSlice`, `stringBytes`.
- [ ] Outside `changes/archive/`, confirm no *use* of `m[k]`, `m[k] = v`, or
      `m.key` remains in the language chapters (the Map section states they do
      not exist, and the `E0215` row uses `m[k]` as a rejected example);
      `xs[i]` stays, and the *Absent map key* row is gone.
- [ ] Re-read the prelude example, the `assert` example, the testing example,
      the `for` iteration bullet, and the Map/List/Set sections against the
      new members.

## Downstream

- [ ] Update the standard-library specification's references to the old
      names: `modules/fs.md` (`listSize`, `stringBytes`), `modules/process.md`
      (`listSize`), `modules/regex.md` (`stringSize`, `stringCodeAt`,
      `stringSlice`), `modules/README.md` (`stringBytes`); check any Map access
      in the std chapters.
- [ ] Implementation: add the closed built-in member sets for `List`, `Map`,
      `Set`, and `string`; the List dynamic members (`push`/`pop`/`shift`/
      `unshift`/`splice`/`resize`/`truncate`/`clear`/`slice`/`indexOf`/
      `lastIndexOf`/`includes`/`join`/`reverse`), the synchronous callback
      bridge and `map`/`filter`/`find`/`findIndex`/`some`/`every`/`reduce`/
      `sort`, and their bounds/length failures; properties
      `.len`/`.size`/`.byteLen`; static
      `Map::fromEntries` and `Set::fromList`; remove the Map/Set subscripts
      and the Map member form; remove the old prelude operations; remove the
      *Absent map key* runtime trap; update `E0215`, `E0102`, `E0204`,
      `E0402`, `E0404`, and `E0212` coverage.
- [ ] Check `xulo-spec/machine-readable.md` and the grammar for stale Map
      access references (none expected: the subscript grammar is unchanged,
      and the restriction is a checking rule).

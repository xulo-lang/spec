# Align Intrinsic Casing

**Status:** draft · **Date:** 2026-10-17 · **Affects:** `builtins/prelude.md`,
`builtins/intrinsic-functions.md`, `types/traits.md`, `testing.md`

## Summary

Rename the multi-word prelude intrinsics and the `ToString` method from
snake_case to lowerCamelCase, the convention `names.md` already recommends
for functions and methods:

| Old | New |
|-----|-----|
| `map_size` | `mapSize` |
| `map_has_key` | `mapHasKey` |
| `map_get` | `mapGet` |
| `map_remove` | `mapRemove` |
| `map_to_entries` | `mapToEntries` |
| `map_from_entries` | `mapFromEntries` |
| `set_from_list` | `setFromList` |
| `set_insert` | `setInsert` |
| `set_remove` | `setRemove` |
| `set_contains` | `setContains` |
| `set_size` | `setSize` |
| `list_size` | `listSize` |
| `list_push` | `listPush` |
| `list_remove` | `listRemove` |
| `to_string` (method of `ToString`) | `toString` |

The single-word names — `print`, `println`, `assert`, `panic`, `str`,
`enumerate`, `timeout`, `race`, `resolve`, `all` — already follow the
convention and are unchanged.

```xulo
let m = { a: 1 }
print(mapGet(m, "z"))     // null: absent, and no failure
let s = setFromList([1, 2])
```

The deltas in [`specs/`](specs/) are the text the chapters gain once the
change is accepted.

## Motivation

### The spec teaches what it forbids

`names.md` recommends lowerCamelCase for functions, and every multi-word
example in the chapters follows it — except the prelude itself. `testing.md`
was until recently teaching `map_get(…)` beside a convention table that
names `sumTo`. A reader who trusts the prelude as a model learns the wrong
style from the language's own textbook.

### One style, not two

The single-word intrinsics (`print`, `assert`) already satisfy
lowerCamelCase, so today the prelude mixes two function styles for no
semantic reason. The rename makes the surface rule what `names.md` says it
is: lowerCamelCase, one style, everywhere.

## Scope

**In scope.** The fifteen multi-word names in the table above and the
`ToString` method, across their declarations, prose, and examples.

**Out of scope.** Single-word intrinsics; built-in type names, protocol
names, and namespaces (all UpperCamelCase and unchanged); core syntax that
replaces functions (`m[k]`, `+`); converting the operations to methods.

## Compatibility

The rename is a source-level break for any program using the old spellings;
the replacement is mechanical. The prelude's reserved names are reserved by
reference to its declarations
([`../../builtins/prelude.md`](../../builtins/prelude.md), Reserved names), so the
reserved set shifts with the rename automatically: `map_get` becomes an
ordinary legal name and `mapGet` becomes reserved. No diagnostic message
quotes an intrinsic name, and no rule other than spelling changes.

## Non-goals

- No semantic or signature change: `mapGet` is exactly `map_get`.
- No method conversion (`m.get(k)`); the operations remain free functions.

## Readiness

- [x] Depends on no unaccepted change; the deltas are written against the
      current chapters.
- [x] Every declaration, prose mention, and example is covered by
      [`specs/`](specs/) and [`tasks.md`](tasks.md).

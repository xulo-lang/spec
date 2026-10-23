# Change String Introspection

**Status:** accepted · **Date:** 2026-10-23 · **Affects:**
`builtins/prelude.md`, `types/primitive-types.md`,
`expressions/path-and-access.md`, `memory-and-runtime.md`,
`error-handling.md`

## Summary

The specification makes strings non-indexable — `s[i]` is `E0215` — and
tells the reader to "select a character by other means, such as the
built-in string intrinsics," while also saying that inspection operations
"are library-level and out of scope." The promised intrinsics do not
exist anywhere, and the self-hosting bootstrap needs them first: a
compiler written in Xulo measures source text, locates diagnostics, and
renders UTF-8 bytes. This change makes the promise true with four
prelude functions:

> `stringSize(s): int`, `stringCodeAt(s, i): int`,
> `stringSlice(s, start, end): string`, `stringBytes(s): List<u8>` —
> scalar-count inspection, scalar at an index, a scalar-range slice, and
> the UTF-8 bytes.

Indices count Unicode scalar values; `stringCodeAt` yields the scalar's
code point as an `int`. An out-of-range index or slice bounds are a
runtime failure, joining the closed tables. The deltas in
[`specs/`](specs/) are the text the chapters gain once the change is
accepted.

## Motivation

### The spec's own cross-reference dangles

`expressions/path-and-access.md` sends the reader to "the built-in string
intrinsics"; `types/primitive-types.md` says inspection is "library-level
and out of scope for this specification." Both statements cannot be true:
the first names a spec-defined thing that does not exist, and the second
is false of a self-hosting language the spec already equips with prelude
collection operations. Closing the gap means either deleting the promise
or writing the functions. Deleting it leaves the bootstrap without
specified string access — the exact capability Change-1-era decisions put
in the prelude for lists and maps.

### Four small functions are the whole bootstrap's string API

Measuring a source file, slicing out a token, decoding an identifier's
bytes — these are `stringSize`, `stringCodeAt`, `stringSlice`,
`stringBytes`. With them, a self-hosted compiler needs no further
language support for text; without them it needs FFI or an
implementation-specific escape hatch on day one. The collection
operations (`mapGet`, `listRemove`, …) set the precedent: pure,
non-failing-or-explicitly-failing inspection lives in the prelude, not in
the grammar.

### The failure shape is already written

Out-of-range list indices are a closed-table runtime failure;
`mapGet`/`listRemove` are the non-failing lookups. The string functions
take the list side of that split — `stringCodeAt` traps where an
optional-returning variant would answer — and the new condition slots
into both closed tables next to its siblings. No new table, no new
category.

## Scope

**In scope.** The four prelude functions and their semantics (scalar
indices, UTF-8 bytes); the runtime-failure condition *String index out of
range* in both closed tables; the rewritten non-indexability bullets in
`primitive-types.md` and `path-and-access.md`; the reservation of the
prelude names.

**Out of scope.** Indexing syntax on `string` (stays `E0215`); a `char`
type; case conversion, trimming, splitting, or searching (library); any
encoding other than UTF-8; byte-oriented indexing of `string` itself
(bytes are reached via `stringBytes` and a `List<u8>`).

## Compatibility

No conforming program can regress: no construct's meaning changes; four
new prelude names join the reserved set, so a module-scope declaration
that shadowed one of them gains an `E0106` — exactly the treatment every
prelude operation already has, and no program can have relied on
declaring them since they did not exist. `s[i]` on strings keeps its
`E0215` verdict.

## Non-goals

- No `stringToChars` returning a list of code points — `stringCodeAt` in
  a loop is the specified path, and allocation of a per-scalar list is
  not a v1 requirement.
- No `stringFromCode` / decoding constructor — construction from bytes
  and code points belongs with the standard library, where `Result`-typed
  decode failures can be expressed.

## Readiness

- [x] Depends on nothing unaccepted; every delta is written against the
      current, folded chapters.
- [x] All five affected passages are covered by [`specs/`](specs/) and
      [`tasks.md`](tasks.md), including both closed failure tables, which
      MUST stay byte-identical.

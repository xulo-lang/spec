# Design: string inspection in the prelude

## The decision

Four new prelude functions, unqualified and in scope in every module:

| Signature | Meaning |
|-----------|---------|
| `stringSize(s: string): int` | the number of Unicode scalar values in `s` |
| `stringCodeAt(s: string, i: int): int` | the Unicode scalar value at index `i` |
| `stringSlice(s: string, start: int, end: int): string` | the substring of scalars at indices `start … end − 1` |
| `stringBytes(s: string): List<u8>` | the UTF-8 bytes of `s`, in order |

Indices count Unicode scalar values, not bytes or UTF-16 code units, and
`stringCodeAt` yields the scalar's value (its code point) as an `int`.
An index outside `0 … stringSize(s) − 1`, slice bounds outside
`0 … stringSize(s)` with `start > end`, is a runtime failure — one new
row, *String index out of range*, in both closed tables. The names join
the reserved set exactly as the collection operations did.

## Alternatives considered

### Leave it to the standard library — the current wording

**Cost.** The specification's own reference to "built-in string
intrinsics" (`path-and-access.md`) stays false, and the bootstrap — the
first consumer of the frozen language — starts with no specified text
access at all, worse off than list-using code was. The prelude exists
precisely to carry the operations types need but syntax does not; strings
are the one type it skips while the spec promises they are covered.
Rejected.

### Indexing syntax — `s[i]` on strings

**Cost.** `E0215` and the "not indexable" bullet were deliberate v1
decisions: indexing drags scalar-vs-byte questions into the subscript
operator, and the subscript's element-yield rule has no room for a
per-character story. Changing that is a language-surface change, not a
gap-fill. Rejected; `s[i]` keeps `E0215`.

### A `char` type with `s.chars()`

**Cost.** A new value type with its own literals, exhaustiveness role,
and conversions — a chapter, not a paragraph. The bootstrap needs a code
point as a number (`switch` on it, compare it, format it), which
`stringCodeAt`'s `int` gives directly. Rejected for v1.

### Byte indices on `string` — Python-3-style `s[i]` as bytes

**Cost.** Byte-position inspection is what UTF-8 source tools actually
do, but making it the primary API forces every human-facing question —
"How many characters?" — through byte arithmetic. The chosen pair
answers both audiences: scalar indices by default, `stringBytes` +
`List<u8>` subscripts for byte work, with one allocation where byte work
is done. Rejected as the primary API.

### Four prelude functions, scalar indices — chosen

**Why.** It is the smallest surface that fulfills the spec's existing
promise, matches the collection-prelude precedent (reserved names,
non-failing lookup vs trapping subscript split already in the tables),
and covers the bootstrap's three needs — measure, select, decode — in
pure specified code. The checkable deliverable is precise: four
closed intrinsics with total signatures, one new closed runtime-failure
condition, no grammar change.

## Consequences

### Prelude and names

- `stringSize`, `stringCodeAt`, `stringSlice`, `stringBytes` are prelude
  operations; the reserved-names sentence is generalized to cover the
  string section, so redeclaring one at module scope is `E0106`.
- The functions are non-generic (no type parameters — the argument is
  always `string`), unlike the collection operations.

### Typing

- All four are ordinary non-const calls (constant folding does not apply
  to them); arguments are checked by the ordinary call rules.
- `stringBytes` returns `List<u8>`, so the existing list operations and
  subscripts are the byte-access path — no second byte API.

### Runtime

- One new row in each of the two tables that MUST stay identical
  (`memory-and-runtime.md`, `error-handling.md`), placed after *Absent
  map key* so the subscript-family conditions stay together.
- No new failure category: a trap is a panic like every other row.

### Diagnostics

| Code | Change |
|------|--------|
| `E0215` | unchanged — `s[i]` on strings is still the error; its bullet now names the replacement functions |
| `E0106` | unchanged — gains four more reserved names via the prelude rule, no new registration |

### Grammar

None: no production, token, or precedence changes.

### Parity

The change adds two rewritten bullets, one table row, and one prelude
section of reading; in exchange it fulfills a dangling spec reference,
adds four closed specified operations, and adds a closed runtime-failure
condition — Layer 2 and Layer 3 deliverables. No new syntax appears, so
the gate's syntax clause is untouched.

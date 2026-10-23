# Add Reserved Word Production

**Status:** accepted · **Date:** 2026-10-23 · **Affects:** `grammar.md`

## Summary

The `Token` production references `ReservedWord`, but the chapter defines no
such production and the production index does not list it — while the same
chapter promises that "[e]very nonterminal is defined exactly once" and that
the index maps each one to its section
([`../../grammar.md`](../../grammar.md)). This change adds the missing
production, a lexical condition over the reserved list of
[lexical-structure.md](../../lexical-structure.md), and the matching index
entry, restoring the chapter's own invariant.

Until this change is accepted, `ReservedWord` is the one name `Token`
consumes that a parser generated from the grammar must special-case from
prose ([`../../grammar.md`](../../grammar.md), the paragraph after the
Keywords block). The deltas in [`specs/`](specs/) are the text the chapter
gains once the change is accepted.

## Motivation

### The chapter violates its own invariant

`grammar.md` states twice — at the top and in the index introduction — that
every nonterminal is defined exactly once and listed exactly once. The
production index contains 191 names matching 191 definitions; `ReservedWord`
is referenced by `Token` and defined nowhere, and is the single name in
`Token` absent from the index. An implementer who trusts the stated invariant
(automatically, in a grammar pipeline) hits an unresolved nonterminal
immediately.

### The concept already exists in prose

The Keywords section already says what a `ReservedWord` is: a word whose
spelling appears in the reserved list of `lexical-structure.md`, neither a
keyword nor an identifier. The lexical chapter already states that reserved
words are produced as their own tokens
([`../../lexical-structure.md`](../../lexical-structure.md)). Only the EBNF
form is missing; the meaning is fully specified and does not change.

## Scope

**In scope.** One production — `ReservedWord` as a lexical condition over the
reserved list of `lexical-structure.md` — appended to the Keywords block, and
one index name added to the first row of the production index.

**Out of scope.** Any change to the reserved list itself, to the keyword set,
to token classification rules, to `names.md`, or to diagnostics. The prose
paragraph that defines `ReservedWord` in words is left as it stands.

## Compatibility

Purely editorial: no token sequence changes meaning, `let class = 1` remains
ill-formed at the same lexical layer, the 61-keyword count is untouched, and
no diagnostic code or message changes. The grammar becomes self-consistent
for any consumer that reads only `grammar.md`.

## Non-goals

- No move of the reserved list into `grammar.md` (it stays in
  `lexical-structure.md`, the chapter that owns it).
- No reduction or expansion of the reserved set.

## Readiness

- [x] Depends on nothing unaccepted; the delta is written against the current,
      folded `grammar.md`.
- [x] Both edits (Keywords block, index row) are covered by
      [`specs/grammar.md`](specs/grammar.md) and
      [`tasks.md`](tasks.md).

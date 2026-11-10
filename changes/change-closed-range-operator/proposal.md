# Change Closed Range Operator

**Status:** draft · **Date:** 2026-11-10 · **Affects:**
`lexical-structure.md`, `grammar.md`, `ast.md`, `introduction.md`,
`types/primitive-types.md`, `types/enums.md`, `expressions/operators.md`,
`expressions/README.md`, `expressions/control-flow.md`,
`type-system/checking-rules.md`, `builtins/prelude.md`,
`builtins/intrinsic-functions.md`, `components/state.md`,
`error-handling.md`, `memory-and-runtime.md`, `changes/README.md`

## Summary

The closed range is currently spelled `a...b`, while the half-open range is
`a..<b`. This change respells the closed range as `a..=b` and leaves the
half-open range unchanged:

> `a..<b` is half-open and excludes `b`; `a..=b` is closed and includes `b`.

`...` ceases to be a range operator and becomes **only** the spread prefix
inside list and map literals. No diagnostic code is added or removed; the
deltas in [`specs/`](specs/) are the text the chapters gain once the change is
accepted.

## Motivation

### The range pair should mirror the comparison pair

The language already writes its comparisons as `<` and `<=`. A range operator
pair of `..<` and `..=` inherits the same distinction: the trailing mark names
exactly which endpoint is included, because `<` and `=` are already the marks
the language uses for “strictly below” and “up to and including”. The current
pair `..<` / `...` does not have this property: `...` carries no marker at all,
so a reader has to remember that the *absence* of a mark means *inclusive* —
the opposite of the intuition the comparison operators establish.

```xulo
for i in 0..<10 { … }   // 0, 1, …, 9
for i in 0..=9 { … }    // 0, 1, …, 9
```

### One token, two jobs

`...` currently means both “closed range” and “spread prefix”, and the
specification has to disambiguate the two by position — a rule stated at length
in [`lexical-structure.md`](../../lexical-structure.md),
[`grammar.md`](../../grammar.md), and
[`expressions/README.md`](../../expressions/README.md). Separating the
spellings removes the rule: `..=` is an infix range token, `...` is a prefix
that can only begin a list or map element. A parser no longer has to decide,
and a reader no longer has to check whether a `...` sits inside a literal.

The existing typo ``0...=15`` in
[`builtins/intrinsic-functions.md`](../../builtins/intrinsic-functions.md),
[`error-handling.md`](../../error-handling.md), and
[`memory-and-runtime.md`](../../memory-and-runtime.md) is evidence that the
dual role invites confusion: it appears to splice the closed-range `...` and
the inclusive `=` into a spelling the grammar never defines.

### The break is small and mechanical

Every use of the closed-range operator is a two-operand `a...b`; the change is
a token substitution in the chapters and their examples. There is no semantic
change to `Range<T>`, iteration, or range patterns — only the spelling of the
closed form. The spread `...` is untouched, so collection literals keep
working as they are.

## Scope

**In scope.** The spelling of the closed-range operator (`...` → `..=`), the
operator inventory and longest-match rules, the grammar and AST operator names,
the range-pattern production, and every example that writes a closed range.
Also the three ``0...=15`` typos, which become ``0..=15``.

**Out of scope.** Half-open `..<` (unchanged), the `Range<T>` type and its
iteration semantics, spread syntax and its typing, and any new diagnostic. The
archived change directories are historical records and are not rewritten.

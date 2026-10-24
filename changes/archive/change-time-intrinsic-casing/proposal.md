# Change Time Intrinsic Casing

**Status:** accepted · **Date:** 2026-10-24 · **Affects:** `names.md`,
`builtins/intrinsic-functions.md`

## Summary

`names.md` says functions, methods, and variables are lowerCamelCase —
as a recommendation only. Two built-in members break it:
`Time::now_nanos` and `Time::monotonic_nanos`. The casing sweep that
lowercased every multi-word prelude intrinsic (`mapGet`, `listRemove`)
missed the `Time` namespace, so the specification's own examples now
teach one thing and spell two functions another.

This change makes the recommendation **normative for official names**
and fixes the two that violate it:

> The conventions in the table are normative for every name this
> specification defines itself — a built-in function or method is always
> lowerCamelCase, a built-in type always UpperCamelCase. User code
> keeps the recommendation, with no diagnostic.

> `Time::now_nanos` → `Time::nowNanos`, `Time::monotonic_nanos` →
> `Time::monotonicNanos`.

The standard library is documented outside this specification but is
held to the same table (informative cross-reference). The deltas in
[`specs/`](specs/) are the text the chapters gain once the change is
accepted.

## Motivation

### The specification must obey its own style

A reader meets `lowerCamelCase` in `names.md`, then meets
`monotonic_nanos` in the very next chapter family. Two conforming
implementations of the prelude would disagree on the *spelling* of a
member — which is an API-visible difference, not a style question: the
name is part of the call. Leaving built-ins exempt from the table means
the table does not describe the language's own surface.

### The precedent already renamed the others

`align-intrinsic-casing` moved every multi-word prelude intrinsic to
lowerCamelCase (`mapFromEntries`, `setInsert`, `toString`). `Time::`
carried two stragglers through that sweep. Completing it is the same
edit in the same file.

### User code stays free

Nothing here adds a diagnostic or constrains program names: the
RECOMMENDED paragraph for user code is untouched. The change only draws
the line the title already implies — the official surface is official
spelling.

## Scope

**In scope.** The normative official-names sentence in `names.md`; the
renames `now_nanos` → `nowNanos` and `monotonic_nanos` →
`monotonicNanos` everywhere they appear in
`builtins/intrinsic-functions.md` (Time table, two prose bullets, the
elapsed-time example, the complete signature index).

**Out of scope.** Any naming diagnostic for user code; any other
built-in name (all others already comply); the standard library's own
chapters (documented on xulo-website); example-code cleanup elsewhere
in the chapters (a separate, non-normative commit).

## Compatibility

No program can regress: the implementation repository is untouched by
this session, and no Xulo program exists that calls the old spellings —
they were introduced by this same draft specification. The rename is a
breaking rename only against unreleased drafts.

## Non-goals

- No enforcement mechanism (no `E` code, no lint).
- No rename of `Time::monotonic` / `Time::now` (already compliant) or
  of the unit suffix policy for future members — future members simply
  follow the table.

## Readiness

- [x] Depends on nothing unaccepted; every delta is written against the
      current, folded chapters.
- [x] Both affected passages are covered by [`specs/`](specs/) and
      [`tasks.md`](tasks.md).

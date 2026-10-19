# Add Test Title

**Status:** accepted · **Date:** 2026-10-16 · **Affects:** `grammar.md`,
`testing.md`, `types/traits.md`, `ast.md`

## Summary

A test declaration MAY carry a display title in a parenthesized string
between the attribute and the signature — `#Test("parse test")`:

```xulo
#Test("parse test")
fn testParse() {
  assert(parseOk(), "parses")
}

#Test
fn mapGetAnswersNull() {
  assert(true, "runs only under a tool")
}

#Test()
async fn delayedTaskFinishes() {
  await Time::sleep(1)
}
```

The title is metadata the declaration carries for a tool's reporting: it
binds no value, enters no namespace, and is never a name. The identifier
remains the test's identity — every reference to it is still `E0107`. The
bare `#Test` stays the canonical, **RECOMMENDED** form when no title is
wanted; `#Test()` is an accepted synonym meaning no title.

Until this change is accepted, `TestDecl` has no parenthesized group
([`../../../testing.md`](../../../testing.md)): `#Test("…") fn` does not parse, and
`#Test() fn` parses as an ordinary `fn` carrying the attribute and is
diagnosed `E0110`. The deltas in [`specs/`](specs/) are the text the chapters
gain once the change is accepted.

## Motivation

### An identifier cannot hold a title

Function identifiers follow the casing conventions of
[`../../../names.md`](../../../names.md) — snake_case, no spaces, no punctuation.
"parse test" cannot be an identifier, and `testParse` is a different thing:
the machine name a tool keys on, not the phrase a human reads in a report.
The title separates the two, exactly as `test('parse test', …)` does in Jest
and `test('name', …)` in Dart.

### The declaration already has a place for it

The test form is a registered attribute shape with an optional parenthesized
group in the generic attribute grammar. Giving `#Test` an optional string
group makes the form's spelling complete: the same parentheses that carry
derive targets for `#Derive` carry the title for `#Test`, each fixed by its
registry entry.

## Scope

**In scope.** The `TestDecl` production gains an optional
`("(" [ StringLiteral ] ")")` group; the registry entry fixes its shape;
`TestDeclaration` gains a `title` field; the testing chapter specifies what
the title is and is not.

**Out of scope.** Titles on any other declaration; interpolation or
non-literal payloads; uniqueness, discovery, ordering, and report formatting
— all tool properties, unchanged.

## Compatibility

Existing tests are untouched: the bare form parses exactly as before.
`#Test("…") fn` moves from a parse error to well-formed. `#Test() fn` moves
from `E0110` to well-formed with no title. No diagnostic message shape
changes except the `E0110` instance for the test form, which grows the
optional title group.

## Non-goals

- No per-test metadata beyond the title (`#Test(timeout: …)` etc.).
- No effect on `E0107`, on inertness, or on pass/fail.

## Readiness

- [x] Depends only on the folded attribute registry; the deltas are written
      against the current chapters.
- [x] Grammar, registry, AST, and every touched passage are covered by the
      deltas in [`specs/`](specs/) and [`tasks.md`](tasks.md).

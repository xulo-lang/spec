# Change Test to Attribute

**Status:** draft · **Date:** 2026-10-16 · **Affects:** `testing.md`,
`grammar.md`, `lexical-structure.md`, `type-system/errors.md`,
`type-system/checking-rules.md`, `modules/source-files.md`,
`types/traits.md`

## Summary

The test declaration is spelled with the declaration-attribute mechanism
instead of a reserved keyword. `test [async] fn` becomes `#Test [async] fn`:

```xulo
#Test
fn map_get_answers_null() {
  let m = { a: 1 }
  assert(map_get(m, "z") == null, "absent key is null")
}

#Test
async fn delayed_task_finishes() {
  await Time::sleep(1)
}
```

The declaration keeps everything it has today: file scope, an empty parameter
list, no return type, no generics, no `pub`, an inert identifier whose any
reference is `E0107`, and the same checking, `async`, and pass/fail rules.
The `test` keyword leaves the closed keyword set (42 → 41) and becomes an
ordinary identifier — **not** reserved — exactly as `from` and `import`
did when the import syntax was retired.

Until this change is accepted, the form is `test [async] fn name() { … }`
with `test` a keyword ([`../../testing.md`](../../testing.md)), and `#`
carries no test meaning. This change also requires
[add-declaration-attributes](../add-declaration-attributes/proposal.md) to be
accepted and folded first: `#Test` is an entry in the attribute registry that
change introduces.

## Motivation

### One marker mechanism, not two

`#Derive(ToString)` (from `add-declaration-attributes`) marks a declaration
with a checker-interpreted name. The test form marks a declaration with a
keyword. The two mechanisms sit side by side for no semantic reason — both
are inert, declaration-level marks that no execution path can strip. With
the attribute registry in place, `#Test` is the same kind of mark as
`#Derive`, and the language has one way to write "this declaration carries a
name the checker interprets".

### The keyword set is a cost with no return

Every keyword is a word every program loses as an identifier and every
reader must special-case in expression position. `test` is the only keyword
whose entire job is "mark this declaration" — it declares nothing, binds
nothing, and appears nowhere else. An attribute spends no keyword.

### Rust's precedent, spelled the Xulo way

Rust marks tests with `#[test]`, and the ecosystem reads that as "a test"
at a glance. Xulo already diverges from Rust's `#[…]` by using `#Name`
attribute syntax; `#Test` is the same idea in Xulo's own spelling.

## Scope

**In scope.** The `TestDecl` production is re-spelled; the keyword table and
keyword production lose `test`; the attribute registry gains a `Test` entry
with its placement and shape rules; the diagnostics `E0107`, `E0109`,
`E0110`, and `E0501` are re-worded where they mention the old spelling; the
chapters that name the form (`testing.md`, `source-files.md`,
`checking-rules.md`) are updated.

**Out of scope.** No semantic change: discovery, checking, inertness, the
`async` marker, and pass/fail are untouched. No new attributes (`#Bench`,
`#Ignore`, …) are registered. The AST `TestDeclaration` node is unchanged —
the swap is purely syntactic, and the node type still carries test-ness.

## Compatibility

Every existing `test fn` / `test async fn` declaration is a one-for-one
rewrite to `#Test fn` / `#Test async fn`:

```xulo
// before
test fn map_get_answers_null() {
  assert(true, "runs only under a tool")
}

test async fn delayed_task_finishes() {
  await Time::sleep(1)
}

// after
#Test
fn map_get_answers_null() {
  assert(true, "runs only under a tool")
}

#Test
async fn delayed_task_finishes() {
  await Time::sleep(1)
}
```

The old spelling is a parse error after the change — `test` is an ordinary
identifier, and `test fn …` does not parse as any declaration — so no program
is silently reinterpreted. The name `test` itself becomes usable as an
identifier wherever an identifier is allowed.

## Non-goals

- No `#Bench`, `#Ignore`, or other test-adjacent attributes.
- No change to how a tool discovers or runs tests.
- No attribute list on the test form itself (`#Derive #Test fn` is not a
  test; it is an ordinary `fn` carrying two attributes, and `Test` on a
  plain `fn` is `E0110`).

## Readiness

- [x] [add-declaration-attributes](../add-declaration-attributes/proposal.md)
      must be accepted and folded first; this change's deltas are written
      against the post-attribute chapters and are re-based at acceptance.
- [x] Grammar, registry, diagnostics, and every `test` mention across the
      chapters are covered by the deltas in [`specs/`](specs/) and the
      sweep list in [`tasks.md`](tasks.md).
- [x] Design alternatives — keep the keyword; attribute on an ordinary
      `fn` — are weighed in [`design.md`](design.md) and rejected.

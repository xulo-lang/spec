# Change Attribute Spelling

**Status:** draft · **Date:** 2026-11-05
**Change ID:** `change-attribute-spelling`
**Affects:** `lexical-structure.md`, `grammar.md`, `ast.md`,
`types/traits.md`, `testing.md`, `type-system/errors.md`,
`type-system/checking-rules.md`, `modules/source-files.md`,
`documentation-comments.md`, `builtins/prelude.md`,
`builtins/intrinsic-functions.md`, `expressions/async-expressions.md`,
`introduction.md`, `changes/README.md`, `changes/add-macro-system`

## Summary

Declaration attributes move from the Xulo spelling `#Name(args)` to Rust's
lowercase bracket spelling `#[name(args)]`:

```xulo
#[derive(Debug)]
struct User { name: string, age: int }

#[test]
fn parses_a_key() {
  assert(true, "runs only under a tool")
}
```

Function-like macro calls already use `name!(…)` and do not change. After
this change, both kinds of macro use site read the Rust way: attributes and
derives are `#[…]`, function macros are `xx!()`.

The registered attribute names are lowercased with the brackets:
`#[derive(Debug)]`, `#[test]`, `#[macro(…)]`, and user
attributes such as `#[route(…)]`. Derive targets are type names and keep
their spelling: `#[derive(Debug)]`, not `#[derive(debug)]`.

`derive` and `macro` are reserved words in Xulo. They stay reserved
everywhere and are accepted only as attribute names between `#[` and `]`; this
change does not make them ordinary identifiers. `test` is not reserved, so
`#[test]` needs no exception. The old `#Name` forms are syntax errors, and
the old capitalized built-in spellings (`#[Derive]`, `#[Test]`, `#[Macro]`)
are not registered names.

## Motivation

### The completed macro system should use bracket attributes

The draft `add-macro-system` change adds attribute macros and `name!(…)`
function macros. It currently uses the old `#Name` spelling for the first, so
the completed macro story would read:

```xulo
#route(GET)
fn list_users() { … }

let table = sql!(SELECT * FROM users);
```

The second line is Rust's function-macro shape; the first is not Rust's
attribute shape. The user-facing request is that macro use be written as in
Rust: `#[…]` for attributes and derives, `xx!()` for function macros, with
lowercase attribute names such as `#[test]`. One of those two already
holds; this change makes the other hold too.

### `#Test` is the same marker with an older spelling

`#Test` and `#Derive` are both inert declaration marks interpreted by the
checker. `change-test-to-attribute` made the test declaration use the
attribute mechanism and deliberately kept Xulo's `#Name` spelling, noting
that “Xulo already diverges from Rust's `#[…]`.” That divergence is now the
only reason the macro story has two spellings. Moving the marker to `#[…]`
and the names to lowercase makes the attribute family read as Rust does
without changing what any attribute means.

### Reserved words do not block lowercase names

Rust spells the attributes `#[derive]` and `#[test]`. Xulo reserves `derive`
and `macro`; lowercasing them naively would put a reserved word where the
grammar expects an identifier. The fix is not to unreserve them — that would
make `fn derive()` and `let macro = …` legal everywhere — but to give the
attribute name its own production and namespace. `AttributeName` accepts an
ordinary identifier plus the two reserved spellings; the AST carries an
`AttributeNameNode` with a string, not an `IdentifierNode`. `test` is already
an ordinary identifier, so `#[test]` uses the identifier arm.

## Scope

**In scope.** The `Attribute` and `TestDecl` productions; the new
`AttributeName` production; the lowercase registered names `derive`, `test`,
and `macro`; every declaration attribute occurrence in the chapters, examples,
and tests; the `AttributeNameNode` in the AST; diagnostic examples and
message shapes that quote the old spelling. The test title is removed:
`TestDeclaration` no longer carries a title and `#[test]` takes no arguments.

**Out of scope.** Unreserving `derive` or `macro`; any uppercase or bracketless
aliases; changing derive targets (`Debug` stays `Debug`); attribute
arguments for other attributes, registry membership, placement, or argument
shapes beyond removing the test title; derive expansion; test checking,
discovery, or execution; function macro syntax (already `name!(…)`); the closed
`@State`/`@Store`/`@Effect`/`@Environment` component family; any new
attribute.

## Compatibility

This is a breaking source change against unreleased drafts. The rewrite is
mechanical except that test titles are dropped:

| Before | After |
|--------|-------|
| `#Derive(Debug)` | `#[derive(Debug)]` |
| `#Test` | `#[test]` |
| `#Test()` | `#[test]` |
| `#Test("parse test")` | `#[test]` (the title is removed) |
| `#Macro(kind = "attribute")` | `#[macro(kind = "attribute")]` |
| `#route(GET)` | `#[route(GET)]` |

The `macro` row uses the argument shape introduced by `add-macro-system`;
this change only re-spells the attribute name. Test titles are dropped by this
change: a test that carried a title keeps its identifier and body, and the
title is removed. `#[test()]`, `#[test("…")]`, and `#[test(name, …)]` are not
test declarations; they fall under `E0110` when the attribute argument syntax
accepts them. This supersedes the optional title accepted by
`archive/add-test-title`.

The old forms are parse errors, never reinterpreted. An old capitalized
bracket spelling such as `#[Derive]` parses as an attribute name but is not
registered, so it is `E0108` unless a user attribute of that spelling is in
scope. A compiler MAY emit a migration hint pointing at the lowercase
bracket spelling, but the hint introduces no new language error code.

The draft `add-macro-system` change MUST be rebased onto this spelling before
it is accepted. Its semantics — sandbox, input records, expansion order,
quota, cache — do not change.

## Non-goals

- No uppercase aliases (`#[Derive]`, `#[Test]`, `#[Macro]`).
- No unreserving of `derive` or `macro`, and no change to the reserved-word
  list.
- No change to derive targets, macro argument names, or context API names.
- No change to `@` component declarations or to the `$` binding prefix.
- No new attributes, derive targets, or macro capabilities.
- No compatibility/deprecation window for `#Name`.

## Readiness

The design alternatives are resolved in [design.md](design.md): lowercase
bracket names with `AttributeName` handling the reserved spellings. The
authoritative deltas for every affected chapter are under
[`specs/`](specs/); the dependent `add-macro-system` rebase and test updates
are listed in [tasks.md](tasks.md). The change is a draft and does not amend the chapters
until accepted.

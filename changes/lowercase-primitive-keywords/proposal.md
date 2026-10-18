# Lowercase Primitive Keywords

**Status:** draft · **Date:** 2026-10-17 · **Affects:** every chapter that
spells a primitive type (signatures, annotations, examples, prose)

## Summary

The built-in primitive **value** types become lowercase reserved keywords;
every other type name stays UpperCamelCase:

```xulo
let count: int = 42
let ratio: float = 0.5
let name: string = "lyy"
let flag: bool = true
let sizes: Map<string, u32> = { "w": 1920 }

fn Counter(): View { … }          // View stays: the UI marker type
struct Image { width: u32 }       // user types stay UpperCamelCase
```

**The seventeen new keywords** — `null` is already a keyword — with their
current spellings shown for reference:

| New keyword | Was |
|-------------|-----|
| `bool` | `Boolean` |
| `string` | `String` |
| `number`, `int`, `float` | `Number`, `Int`, `Float` |
| `i8` `i16` `i32` `i64` | `I8` `I16` `I32` `I64` |
| `u8` `u16` `u32` `u64` | `U8` `U16` `U32` `U64` |
| `f32`, `f64` | `F32`, `F64` |
| `unit`, `unknown` | `Unit`, `Unknown` |

Each keyword keeps its word and changes only its case, except `bool`, which
also shortens `Boolean`; the keyword count
goes from 41 to **58** in both ledgers (the grammar ledger's prose currently
says 42 against its own 41-entry list — corrected here).

**UpperCamelCase stays** for everything else: `View` (the UI marker type),
`Range<T>`, the built-in composites `List`, `Map`, `Set`, `Task`,
`Result`, `Error`, the protocol `ToString`, and every user-declared type,
trait, enum, and component.

The deltas in [`specs/`](specs/) carry the structural edits — the keyword
table, the shadowing rules, the reserved-names table, the primitive-types
chapter's framing — and the rename of the remaining occurrences across the
chapters is specified by the table above and executed per
[`tasks.md`](tasks.md).

## Motivation

### Lowercase means keyword, everywhere in the ecosystem

In Rust, Go, and TypeScript the scalar type names are lowercase because
they are keywords — closed, reserved, not nameable by user declarations.
Xulo's primitives are the same closed set, but spelled UpperCamelCase, which
in those ecosystems marks *named* types (`String`, `Vec`, `Promise<T>`).
The new spelling puts Xulo's types in the column a reader coming from any
of those languages expects, and reserves the capital for names a program
can declare.

### The capital now carries information

After this change a capitalized type name in a signature is always a name
with an identity — a struct, an enum, a generic instantiation, `View` —
never a scalar. `Map<string, int>` and `Map<string, Image>` read at a
glance: the first argument is a primitive, the second is somebody's type.

### One rule

Every type name follows a single case rule keyed on one question — *can a
program declare this?* No → lowercase keyword. Yes → UpperCamelCase. No
third class.

## Scope

**In scope.** The seventeen keyword additions; every occurrence of the
eighteen primitive names (including the existing `null`) in type position,
code spans, and code blocks across the spec; the keyword ledgers; the
shadowing and reserved-names rules that the keywordization changes.

**Out of scope.** `View` and `Range<T>` — primitive but UpperCamelCase, for
the reasons in the design; the built-in composites and protocols; AST node
names (`StringLiteral`, `NumberLiteral` are spec-internal identifiers, not
source spellings); any semantic change to the types themselves.

## Compatibility

Existing programs rewrite mechanically: the rename table is the whole
migration. The seventeen spellings become reserved, so a program that used
one as an identifier — `let int = 0`, `fn view(): …` for a *function* named
`view` — no longer compiles; that is the intended effect. Shadowing of the
primitive names, defined today in `names.md`, ends with them: they are
keywords, and keywords were never declarable. The reserved-name diagnostic
`E0106` keeps its mechanism and loses its base-type row.

## Non-goals

- No change to the numeric tower, coercions, or any typing rule — only the
  spellings.
- No method or namespace reorganization of the prelude
  (`align-intrinsic-casing` is independent and compatible).

## Readiness

- [x] Depends on no unaccepted change; the deltas are written against the
      current chapters.
- [x] Structural edits are in [`specs/`](specs/); the mechanical rename is
      fully specified by the table above and checked by the sweeps in
      [`tasks.md`](tasks.md).

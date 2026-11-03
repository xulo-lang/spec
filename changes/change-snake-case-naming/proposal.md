# Change Snake-Case Naming

**Status:** draft · **Date:** 2026-11-03 · **Affects:** `names.md`,
`builtins/prelude.md`, `builtins/intrinsic-functions.md`, `builtins/README.md`,
`types/traits.md`, `types/composite-types.md`, `types/primitive-types.md`,
`functions.md`, `expressions/path-and-access.md`,
`type-system/checking-rules.md`, `type-system/errors.md`, `concurrency.md`,
`error-handling.md`, `memory-and-runtime.md`,
and the example code throughout the chapters

## Summary

Xulo spells the names of functions, methods, variables, parameters, and
struct fields with `lowerCamelCase`. This change replaces that convention
with `snake_case`:

> A word boundary in a name is written `_`. The spelling lowercases every
> word of the name and joins the words with a single underscore; an acronym
> is spelled as its lowercase letters, and a digit stays attached to the
> word it belongs to. An ordinary name carries no leading, trailing, or
> doubled underscore.

The change keeps the two distinctions the language already draws. Types stay
UpperCamelCase, so a capital initial still means "type" and a lowercase
initial still means "value or function". Constants stay UPPER_SNAKE_CASE.
A parameter declared as a component attribute stays lowerCamelCase even
though it is still written and matched as a named argument; every other
named parameter's label is the parameter's own name and therefore follows
the snake_case parameter rule.

The conventions remain RECOMMENDED for user code: no diagnostic is added,
and no program is rejected for spelling a name either way. The conventions
remain normative for the names this specification defines itself, so every
multi-word official name moves with the table:

| Kind | Old | New |
|------|-----|-----|
| `Math` member | `wrappingAdd` | `wrapping_add` |
| `Math` member | `wrappingSub` | `wrapping_sub` |
| `Math` member | `wrappingMul` | `wrapping_mul` |
| `Math` member | `popCount` | `pop_count` |
| `Math` member | `countLeadingZeros` | `count_leading_zeros` |
| `Math` member | `countTrailingZeros` | `count_trailing_zeros` |
| `Math` member | `rotateLeft` | `rotate_left` |
| `Math` member | `rotateRight` | `rotate_right` |
| `Math` member | `toFixed` | `to_fixed` |
| `Time` member | `nowNanos` | `now_nanos` |
| `Time` member | `monotonicNanos` | `monotonic_nanos` |
| `Display` action | `toString` | `to_string` |
| `List`/`Array` member | `indexOf` | `index_of` |
| `List`/`Array` member | `lastIndexOf` | `last_index_of` |
| `List`/`Array` member | `forEach` | `for_each` |
| `List`/`Array` member | `findIndex` | `find_index` |
| `List::splice` parameter | `deleteCount` | `delete_count` |
| `Map` static member | `fromEntries` | `from_entries` |
| `Set` static member | `fromList` | `from_list` |
| `string` member | `byteLen` | `byte_len` |
| `string` member | `codeAt` | `code_at` |
| Service method | `onMessage` | `on_message` |

Single-word names — `len`, `size`, `get`, `print`, `abs`, `now`, `filled`,
and the rest — are unchanged. `atan2`, `log2`, and `log10` keep their digit
suffix, because the digit is part of the word rather than a boundary.

```xulo
fn sum_to(limit: int, step_size: int): int {
  let mut total = 0
  let mut i = 0
  while i < limit {
    total = total + i
    i = i + step_size
  }
  total
}

let user_name = "ada"
let counts = { "a": 1, "b": 2 }
for key in counts {
  print(key)
}
```

The deltas in [`specs/`](specs/) are the text the chapters gain once the
change is accepted.

## Motivation

### An underscore is an explicit word boundary

English separates words with spaces; a compound identifier has no spaces, so
a reader has to infer the boundary from a capital letter. `countLeadingZeros`
and `countTrailingZeros` differ in the middle of a long unseparated run, and
`nowNanos`, `isPub`, and `byteLen` each hide a boundary at a different
position. `_` states the boundary directly: `count_leading_zeros`,
`count_trailing_zeros`, `now_nanos`, `is_pub`, `byte_len`. That is the same
reason the language already spells constants `APP_NAME` rather than
`appName`.

### The official surface should teach one style

Every chapter example and every built-in name is part of what a reader
learns from. As long as the table says lowerCamelCase but the underlying
words are opaque runs, the specification teaches a style that makes its own
identifiers harder to scan. Moving both the recommendation and the official
names together keeps the specification self-consistent: the table, the
prelude, and the examples all show the same spelling.

### The type/value distinction survives

A blanket "everything is snake_case" rule would collide a type `User` with a
variable `user`, and a component `Text` with a local `text`. Keeping
UpperCamelCase for types preserves the visual distinction the language
already relies on: `let user: User = User(...)` reads unambiguously, exactly
as `let counts: Map<string, int> = Map::from_entries(...)` does.

## Scope

**In scope.**

- The `names.md` convention table and its surrounding paragraph: functions,
  methods, variables, parameters, and struct fields become snake_case.
- The official-name sentence and every multi-word official name it governs:
  the intrinsic functions in `builtins/intrinsic-functions.md`, the built-in
  collection and string members in `types/composite-types.md` and
  `types/primitive-types.md`, the derived `to_string` action in
  `types/traits.md` and `builtins/prelude.md`, and the references to those
  names in `builtins/README.md`, `functions.md`,
  `expressions/path-and-access.md`, `type-system/checking-rules.md`,
  `error-handling.md`, and `memory-and-runtime.md`.
- The example code in the chapters, which follows the recommendation.

**Out of scope.**

- Type names, enum variants, protocol names, namespace names, and constants:
  they keep their present spellings (`User`, `Theme::Dark`, `Display`,
  `Math`, `PI`).
- Parameters declared as component attributes: they keep lowerCamelCase
  (`onClick`, `bgImage`, `spacing`, `variant`) because the component's
  contract, not the language, defines them as attributes. Every other named
  parameter's label is its parameter name and follows the parameter rule, so
  `delete_count` is also the label used at a call site.
- The AST schema in `ast.md`: its field names describe a language-agnostic
  data structure, not Xulo source identifiers. The documentation-comment
  JSON schema in `documentation-comments.md` is out of scope for the same
  reason.
- The grammar and the machine-readable grammar files: the `Identifier`
  production already admits `_`, so no grammar token changes.
- Any naming diagnostic or formatter rule: the conventions remain
  RECOMMENDED for user code.

## Compatibility

The rename is a source-level break against unreleased drafts only; no Xulo
program exists that uses the old official spellings, and user code is not
checked either way. The reserved-name set shifts with the rename
automatically because it is defined by reference to the prelude's
declarations: `map_get`-style spellings are already ordinary names, and
`to_fixed` becomes reserved while `toFixed` becomes an ordinary legal name.
No diagnostic message quotes a renamed intrinsic, and no rule other than
spelling changes.

This change reverses the casing direction of two accepted changes,
[`archive/align-intrinsic-casing`](../archive/align-intrinsic-casing/proposal.md)
and [`archive/change-time-intrinsic-casing`](../archive/change-time-intrinsic-casing/proposal.md).
It keeps the design both established — the specification's own names follow
its table — and changes only which case the table names.

## Non-goals

- No enforcement mechanism: no `E` code, no lint, no formatter rule.
- No change to the `Identifier` grammar, to name resolution, or to
  shadowing.
- No rename of types, variants, namespaces, or constants.
- No conversion of free functions to methods, or the reverse.

## Readiness

- [x] Depends on no unaccepted change; the deltas are written against the
      current chapters.
- [x] Every normative name and every chapter that quotes one is covered by
      [`specs/`](specs/); the example sweep is listed in [`tasks.md`](tasks.md).

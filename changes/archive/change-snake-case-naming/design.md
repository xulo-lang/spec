# Design

## The question

The identifier grammar admits ASCII letters, digits, and `_`. Nothing in the
grammar decides how a multi-word name is spelled, so the choice is a
convention: how should a reader recover the words inside `countLeadingZeros`,
`nowNanos`, or `isPub`?

## Alternatives

### A. Keep lowerCamelCase

The status quo. Capital letters mark boundaries, which is familiar from
JavaScript, Java, Swift, and the UI frameworks a UI-first language usually
sits beside. The cost is that a capital letter does two jobs at once — it
marks a boundary and it marks a type name in this language — and that a run
of capitals is ambiguous: `isPub`, `isAsync`, and `URLParser` need the
reader to know where the words start. For a specification that wants its own
examples to be scannable, the boundary is implicit exactly where it matters.

### B. snake_case for functions, methods, variables, parameters, and fields

The chosen design. `_` marks the boundary explicitly, and capitals are left
to do the one job they already do in Xulo: distinguish a type from a value.
Constants already use UPPER_SNAKE_CASE, so the underscore is not a new
character in the naming system. Reading a call `Map::from_entries(entries)`
or a property `s.byte_len` needs no boundary inference. The cost is that the
language spells two-word names differently from the UI frameworks its
components resemble, and that a `_` is one more character to type.

### C. snake_case for everything, types included

Rejected. It makes `User` and `user` the same spelling and removes the only
visual cue that separates a type from a value at a use site. It also churns
every enum variant and protocol name for no scanning benefit.

### D. SCREAMING_SNAKE_CASE for everything

Rejected. It removes the type/value distinction as well, and turns ordinary
code into a wall of capitals. Constants keep UPPER_SNAKE_CASE because the
case itself is meaningful there: a constant is read as a compile-time value.

### E. Enforce a convention with a diagnostic

Rejected. Xulo's conventions are RECOMMENDED for user code; a program that
spells a local `userName` is not ill-formed. The language already reserves
enforcement for name shapes that are genuinely ambiguous or dangerous
(keywords, reserved names, a lone `_`). Adding a casing diagnostic would
reject working programs over style, and it would force a formatter rule the
language does not otherwise need.

## Chosen design

### What changes

| Kind | Convention after the change |
|------|-----------------------------|
| Structs, enums, traits, type aliases, components | UpperCamelCase (unchanged) |
| Enum variants | UpperCamelCase (unchanged) |
| Functions, variables, parameters, fields, methods | snake_case |
| Constants | UPPER_SNAKE_CASE (unchanged) |
| Type parameters | A single letter (unchanged) |
| Const parameters | A single uppercase letter (unchanged) |
| Named-argument labels (ordinary parameters) | the parameter's own name, so snake_case |
| Parameters declared as component attributes | lowerCamelCase (unchanged) |

### Spelling rule

A boundary is a single `_`. Every word is lowercase, and an acronym is
spelled as its lowercase letters (`URL` inside a name becomes `url`), so
`fromEntries` becomes `from_entries` and `byteLen` becomes `byte_len`. A
digit stays attached to the word it belongs to: `log2`, `log10`, and
`atan2` are single words and keep their spelling. An ordinary name does not
begin or end with `_` and does not contain `__`; the lone `_` remains the
wildcard and is not an ordinary identifier.

### Why parameters declared as component attributes stay lowerCamelCase

A named argument is always the parameter's own name: it is written before `:`
in a call and denotes the same declaration the parameter does
([`names.md`](../names.md), Identifiers and Named Entities). That is why an
ordinary parameter follows the parameter rule: `splice`'s `delete_count` is
passed as `delete_count: 2`, never `deleteCount:`.

A parameter declared as a component attribute is the exception, and the
exception is decided by the declaration, not by the call syntax. The
component's contract declares `onClick`, `bgImage`, and `spacing` as its
attributes, and those names are the call-site vocabulary of the component
API rather than the ordinary parameter convention. They are still written
and matched as named arguments — the distinction is which declaration
category the parameter belongs to. Renaming them would churn every component
signature and every call site without making a declaration easier to scan,
so the table keeps them lowerCamelCase even though they are named arguments.

### Why types and constants stay

A capital initial is a load-bearing cue in Xulo: `let user: User = User(...)`
and `let counts: Map<string, int> = Map::from_entries(entries)` are
unambiguous because the type is the identifier that begins with a capital.
Constants are read as compile-time values, and UPPER_SNAKE_CASE already
states both facts — boundary and constness — in the case. Neither group
gains anything from the change, and both would lose their visual role.

## Consequences

### Grammar and lexing

None. The `Identifier` production already accepts `_` after the first
letter, so every new spelling is an identifier under the existing token
rules. `_` alone stays the wildcard, and the name-shape rejections —
keywords, reserved words, lone `_` — are untouched
([`lexical-structure.md`](../lexical-structure.md)).

### Typing and resolution

None. No name changes scope, shadowing, or resolution order. The reserved
set moves only because it is defined by reference to the declarations whose
spellings change; the rules that use it are unchanged.

### Diagnostics

No new code and no edited message. The error codes that mention names by
category (`E0101`, `E0102`, `E0106`) do not quote the casing of a name.

### Migration

A mechanical rename for the specification's own surface. Every occurrence of
a renamed official name is listed in [`specs/`](specs/) or in
[`tasks.md`](tasks.md); the example sweep is the only part that touches
multiple chapters, and it follows the same word-boundary rule.

## Rejected details

- **Underscore as a general separator for types too.** Rejected with
  alternative C: it erases the type/value distinction.
- **Leading or trailing underscore conventions for private names.** Xulo has
  `pub`, modules, and `impl` visibility; it does not need a spelling
  convention for privacy, and adding one would conflict with the wildcard
  `_`.
- **Renaming enum variants to snake_case.** Rejected: a variant is a
  constructor of a type, and the table keeps every type-side name
  UpperCamelCase.

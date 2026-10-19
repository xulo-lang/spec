# Design: lowercase primitive keywords

## The decision

One question sorts every type name: **can a program declare this?**

- No → the name is a lowercase keyword: `bool`, `string`, `number`,
  `int`, `float`, `i8`…`u64`, `f32`, `f64`, `null`, `unit`, `unknown`.
- Yes → the name is UpperCamelCase: user types, `List`, `Map`, `Set`,
  `Task`, `Result`, `Error`, `Range<T>`, `View`, `ToString`.

Each primitive keeps its word and flips only its case. `null` was already a
keyword; the other seventeen are added, and both keyword ledgers go from 41
to 58 (the grammar ledger's prose count of 42 is a pre-existing error
against its own 41-entry list, corrected in the same edit).

## Alternatives considered

### `bool` instead of `boolean` (Rust/Go spelling)

**Accepted.** The pure-case rule wanted `boolean` (TypeScript's spelling),
but `bool` is the spelling of two of the three ecosystems, four letters
shorter, and — unlike `boolean` — not an English word, so it carries none
of the prose-collision hazard the rename has to manage elsewhere. One
vocabulary change among seventeen case flips is a price the migration table
pays openly. The rejected alternative was keeping `boolean` for case
uniformity alone; conciseness and ecosystem weight win.

### Keep the primitives UpperCamelCase (Swift's choice)

**Cost.** Swift is the outlier; the ecosystem gravity of Rust, Go, and
TypeScript pulls the other way, and Xulo's own two audiences — scripting
and UI — meet those three languages daily. Rejected.

### Lowercase spellings but keep them ordinary, shadowable names

**Cost.** The worst of both: `let int = 0` would look like a keyword used in
value position (a syntax error in Rust/Go/TS), while being legal; and every
error message about an unexpected `int` would mislead. Lowercase type names
earn their keep by being keywords. Rejected.

### `View` goes lowercase with the rest

**Cost.** `View` is the UI marker — the return type of every component — and
`view` is one of the most common identifiers a UI program declares
(`let view = render(…)`). Keywordizing it would tax every UI example to
serve a uniformity the marker does not need: `View` is a name with an
identity in the type system (components are functions returning it), so
UpperCamelCase is where it belongs under the rule itself. Rejected;
`View` stays.

### `unit`, `unknown`, `null` are not keywords

**Cost.** They are as language-closed as `int` — no program declares a unit
or an unknown type — and leaving them capitalized would open a third class
the rule exists to abolish. `null` is already a keyword, which shows the
spelling works: `let x: null = null` reads as type-then-value, the same
word doing both jobs (TypeScript's `null` does exactly this). Accepted as
keywords.

## Consequences

### Shadowing splits by declaration-ability

`names.md` currently lets every built-in type name be shadowed (with a
SHOULD NOT). After the change the primitive names are keywords — not
declarable, so not shadowable; the paragraph keeps its content for the
composite names (`List`, `Map`, `Set`, `Task`, `Range`, `View`, `Result`,
`Error` — the last two added to the SHOULD-NOT list, which currently omits
them) and for the intrinsics. The `E0101` consequence of shadowing a
composite name is unchanged.

### `E0106` loses its base-type row, not its mechanism

The reserved-names table in `builtins/prelude.md` drops the *Base and
numeric types* row — a keyword is stopped at the lexical layer, long before
a module-scope declaration is checked — and the *Value and marker types*
row loses `Null`, `Unit`, and `Unknown`, keeping `List`, `Map`, `Set`,
`Task`, `Range`, `Result`, `View`, `Error`. `E0106` itself and its `print`
example are untouched.

### The capital becomes informative

A capitalized type in a signature is now always a name with an identity;
a lowercase one is always language-closed. `Map<string, int>` versus
`Map<string, Image>` states the distinction without a second rule.

### The rename's one hazard: English

Lowercase `string`, `number`, `null`, and `unknown` are ordinary English
words the chapters use in prose ("the number of entries"). The rename
therefore touches code blocks, code spans, and type-position prose only;
[`tasks.md`](tasks.md) carries the review list and the sweep that proves
the English survives.

### Independence from `align-intrinsic-casing`

That change renames free prelude functions (`mapGet`); this one renames type
keywords. Neither touches the other's names, and both compose: after both
fold, `Map<string, int>` is a type written with keywords and `mapGet(m, k)`
a prelude call written in lowerCamelCase.

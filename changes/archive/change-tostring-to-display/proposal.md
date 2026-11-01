# Change ToString to Display

**Status:** accepted · **Date:** 2026-11-01 · **Affects:** `builtins/prelude.md`,
`builtins/README.md`, `builtins/intrinsic-functions.md`, `types/traits.md`,
`types/enums.md`, `types/composite-types.md`, `type-system/coercion.md`,
`type-system/checking-rules.md`, `type-system/errors.md`,
`expressions/literals.md`, `names.md`

## Summary

The string form of a value is served by two built-in protocols and one
built-in context type:

```xulo
struct Format {
  precision: int?,   // the precision a directive asked for, or `null`
  alternate: bool    // the `#` alternate form
}

trait Display {
  fn display(self, format: Format): string
}

trait Debug {
  fn debug(self, format: Format): string
}
```

`Display` is **how a type is shown** — it receives the `Format` the consumer
is asking under and returns the string form. `ToString` stops being a trait a
program implements and becomes what it always was from the caller's side: an
**action**, derived from every `Display` impl —

```xulo
v.toString()   ≡   v.display(Format(precision: null, alternate: false))
```

— so no program ever writes two impls for one type, and `${…}`, `str`, and
`v.toString()` keep their exact acceptance sets and their exact outputs; only
the name a program must implement changes. `Debug` is the structural
counterpart: `#Derive(ToString)` re-targets to `#Derive(Debug)`, generating
the same `User(name: Alice, age: 30)` text it always did, and `print`
consults a user type's `Display` first, then its `Debug`
([intrinsic-functions.md](../../builtins/intrinsic-functions.md)).

Until this change is accepted, `ToString` is the built-in hook that `${…}`
and `str` require, `#Derive(ToString)` is the registered derive target, and
the chapters named above are normative. The deltas in [`specs/`](specs/) are
the text the chapters gain once the change is accepted.

## Motivation

### ToString names the result; Display names the manner

`toString` answers "give me a string" — one answer, no context. What a
formatting engine needs is the other question: **how does this type present
itself under a given format?** Today's signature cannot answer it:

```xulo
impl ToString for Temperature {
  fn toString(self): string { … }    // no way to receive `.2` or `#`
}
```

A `Temperature`, a `Money`, or a padded identifier has a *different correct
text* under different precision and alternate flags; `fn toString(self):
string` has one slot for one answer. `Display` carries the `Format`, and the
action stays a call — `toString` is derived from `display` at the default
`Format`, so `${count}`, `str(n)`, and `v.toString()` never learn about
`Format` at all unless the type's author wants them to:

```xulo
impl Display for Temperature {
  fn display(self, format: Format): string {
    let c = self.kelvin - 273.15
    `${Math::toFixed(c, format.precision ?? 1)}°C`
  }
}
```

### The structural rendering is Debug, not ToString

`#Derive(ToString)` today generates exactly Debug's job wearing ToString's
name:

```xulo
#Derive(ToString)
struct User { name: string, age: int }

let s = User(name: "Alice", age: 30)
print(str(s))                    // User(name: Alice, age: 30)
```

`User(name: Alice, age: 30)` is the value's **internal structure** — field
names, declaration order — served through the channel a user-facing message
would use. The same text has a correct name already: it is what a reader
needs when debugging, and it is not what an author means when they write
`Welcome, ${user.name}` territory — the two views deserve two channels.
Re-targeting derive to `Debug` gives the structural form to the trait whose
job it is and leaves `Display` to be written by hand, where a user-visible
text is an author's decision and not a default.

### The formatting contracts must be language-level

The consumer of these contracts — the formatting engine of the standard
library, whose specification lives in another document — needs a
`Display`/`Debug` pair to dispatch verbs against. It cannot own them:
`${…}`, `str`, and `#Derive` are language constructs, and the type rules
that name their protocol must not cite another specification. The boundary
runs the direction the two documents already agree on: this specification
keeps `std/*` out of scope ([`README.md`](../../README.md), § 1.1), and the
standard library's specification records that it cites *this* one for types
and diagnostics — `Result` and `Error` are its standing precedent — never
the reverse. Defining `Display`, `Debug`, and `Format` here keeps every
citation inside that boundary, and the engine becomes a consumer exactly as
`std` members are today consumers of `Result`.

### Nothing about the call sites changes

`v.toString()` keeps working — it is now the derived action. `${user}`
accepts exactly the types it accepted before. The E0212 acceptance set is
unchanged; its message grows a truer name to implement. The migration a
program performs is a rename plus one parameter (see Compatibility).

## Scope

### In scope

- **Prelude contracts.** `Display` and `Debug` as built-in protocols with
  `fn display(self, format: Format): string` / `fn debug(self, format:
  Format): string`; `Format` as a built-in type
  (`{ precision: int?, alternate: bool }`, default `precision: null`,
  `alternate: false`) with an extension policy — future fields are
  optional-typed, so existing constructions omit them and stay valid;
  `toString` specified as the derived action of every
  `Display` impl; reserved-name rows gain `Format`, `Display`, `Debug`, and
  keep `ToString`.
- **`types/traits.md`.** A built-in formatting-traits section stating the
  contracts — including that `Format` carries no layout (width, alignment,
  padding are applied by the consumer to the returned string, never by an
  implementation) and that a `display` SHOULD reflect `precision`/`alternate`
  and MAY ignore them; `toString` availability and the `E0804` guard against
  declaring it; the derive registry (`Debug` registered, `Display` and
  `ToString` reserved); `#Derive(Debug)` expansion and its field rules.
- **The `str` domain, renamed.** Interpolation, `str`, the coercion row, the
  two checking-rules statements, `List.join`, and the enum bullet say
  `Display` where they said `ToString`; operand sets are unchanged.
- **`print`.** A `struct` or `enum` value consults its `Display` at the
  default `Format`, then its `Debug`; an `enum` with neither keeps the
  table's structural row, and a `struct` with neither — previously
  unspecified — becomes `E0212`. Its `E0212` is timed where the operand's
  type is known (at the call supplying a generic type argument), so
  `print` inside a generic body keeps its signature's openness. A value
  typed `unknown` — as the operand or in any structural position — is
  `E0212` at the call, the error `${…}` already gives it: no rendering
  crosses an `unknown`.
- **Output-table completion.** The rendering table gains the rows its
  promise always implied — `number`, the fixed-bit numerics, `unit`, an
  optional `T?`, `Range<T>`, `Result<T, E>`, tuples, `Error`, `Format`,
  `Ptr` — each mirroring an
  existing convention (`unit`, `Ok(v)`, `(a, b)`, `0..<10`, `<ptr>`). The
  derive's field rule and `print`'s "any type" both need the table total;
  `print(Time::now())` and `print(Ok(1))` are two of the pre-existing
  silences the rows close.
- **Diagnostics text.** `E0109` example, `E0111` note, `E0112` example and
  message, `E0212` trigger/example/message plus its two conditional notes
  (at `print`; for a `Debug` implementor), and the two footnotes after the
  name table. **No new codes**; the `E0801` and `E0805` rows are
  trait-agnostic and unchanged — every other reference to them in the
  chapters is a bound or impl use this change does not touch.
- **`names.md`.** The trait-naming example swaps `Display` for `Clone` —
  `Display` becomes a reserved prelude protocol and may no longer exemplify
  a name a program chooses.

### Out of scope

- **Template format specifiers.** `` `${x:.2}` `` is a template-grammar
  extension with its own design; when it lands it desugars onto `Format` and
  this change's contracts. This change touches the template grammar not at
  all (there is no grammar delta).
- **The standard library's formatting module.** Verbs, the directive
  grammar, width/precision application, and its failure model are the other
  specification's text; after acceptance it cites `Display`/`Debug`/`Format`
  from here. The dependency is recorded on that side (its § 1.3 list), not
  in these chapters.
- **Verb-extension traits** (a `LowerHex`-style family). The contract surface
  stays `Display` + `Debug` + `Format`; whether a type can opt into a
  library's verb vocabulary is a question for the engine's spec.
- **Sealing `Format`.** Construction stays open: `v.display(Format(precision:
  2, alternate: false))` is legal and useful. Making `Format` constructible
  only by consumers needs an opaque type and buys no safety the field types
  do not already give ([design.md](design.md#the-context-type-is-open)).
- **Removing `ToString` from the reserved names**, or adding a dedicated
  migration diagnostic for `impl ToString` (which fails as an unresolved
  trait, `E0101` — see Compatibility).
- **Base-type rendering.** The shortest-`float` rule, the `NaN`/`Infinity`
  spellings, and the output table's existing rows are untouched apart from
  the two-kinds paragraph above; the new rows are *additions* (In scope),
  written to match the conventions those rows already set.

## Compatibility

- **The one breaking rename, written twice.** A hand-written impl

  ```xulo
  impl ToString for User {
    fn toString(self): string { `${self.name} (${self.age})` }
  }
  ```

  becomes

  ```xulo
  impl Display for User {
    fn display(self, format: Format): string { `${self.name} (${self.age})` }
  }
  ```

  The body carries over unchanged unless it wants the context. After the
  change, `impl ToString for …` no longer resolves (`E0101`) — the trait
  name exists only as a reservation.
- **Three names become reserved.** `Format`, `Display`, and `Debug` join the
  prelude's reserved-name table: a module-scope `struct Format`,
  `trait Display`, or `trait Debug` is well-formed today and `E0106` after
  the change. The migration is a rename — which is why the naming chapter's
  own example moves off `Display` to `Clone`
  ([`names.md`](../../names.md)).
- **`#Derive(ToString)` becomes `#Derive(Debug)`.** Before the change it
  works; between drafting and acceptance it is whatever the folded chapters
  say; after, it is `E0111` with the note `` registered targets: `Debug` ``.
  The **rendered output is identical** for fields that render directly — the
  migration is a rename with no visible change.
- **One derive field case moves.** A field whose type had only a hand-written
  `ToString` impl satisfied the old derive; it now needs `Debug` (`E0801`,
  `` `User` does not implement `Debug` ``) — derive it or write it. Fields
  that are containers or enums become *accepted* by the derive where the old
  rule rejected them (`List<int>` no longer fails `#Derive`), because the
  output table renders containers and enum values with no implementation at
  all — and the rule recurses through every structural position (container
  elements, `T?`, tuples, `Result` slots, enum payloads), so `List<User>`,
  `User?`, and `Result<User, string>` each need `User`'s `Debug`.
- **Call sites do not migrate.** `v.toString()`, `${…}`, `str(…)`,
  `.join(…)` accept what they accepted; `E0212`'s message text changes
  (`implement `Display``).
- **`print` of a type with `Display` or `Debug` is now specified** where the
  table had no struct row; a bare struct — previously unspecified behavior —
  becomes a compile-time error rather than an accident. One visible change:
  an `enum` that already had an `impl` used to print by the table's enum
  row and now prints its `Display` (the precedence is uniform for both
  user-declared kinds — the outcomes are pinned in a table in
  [`builtins/intrinsic-functions.md`](../../builtins/intrinsic-functions.md)).
  A second: kinds the table never named — `unit`, the fixed-bit numerics,
  `Range`, `Result`, tuples, `Error`, `Format`, `Ptr` — gain a rendering,
  so `print(Time::now())` moves from unspecified to defined.
- **`print` and `${…}` deliberately diverge for a `Debug`-only type.**
  `print(u)` shows its structure; `` `${u}` `` still demands `Display`
  (`E0212`, whose note for a `Debug` implementor says as much). The
  channels stay separately answerable — structure is
  printable, string form is written — and an author who wants both
  everywhere derives `Debug` beside a hand-written `Display`, the pairing
  this change encourages.

The whole surface in one table:

| Site | Before | After | Visible output |
|------|--------|-------|----------------|
| `` `${x}` ``, `str(x)` | `ToString` | `Display` | unchanged |
| `x.toString()` | a written `impl ToString` | the derived action of `Display` | unchanged |
| `List.join` | `ToString` | `Display` | unchanged |
| `print(x)` — `struct` | unspecified | `Display`, else `Debug`, else `E0212` | from unspecified to defined |
| `print(x)` — `enum` | the table's enum row | `Display`, else `Debug`, else the table's enum row | unchanged without an impl; with `Display`, its display form |
| `print(x)` — built-ins, containers, `null` | the table | the table | unchanged |
| `print(x)` where `unknown` is reached (operand or structural position) | unspecified | `E0212`, the `${…}` error | narrow before printing |
| `#Derive(ToString)` | registered | `E0111`, note `` registered targets: `Debug` `` | — |
| `#Derive(Debug)` | `E0111` | registered | the same bytes as the old derive for fields that render directly |
| module-scope `struct Format`, `trait Display`, `trait Debug` | well-formed | `E0106` | rename the declaration |
| `E0212` | "implement `ToString`" | "implement `Display`", with the notes above where they apply | message text only |

## Non-goals

- **A derivable `Display`.** User-visible text is written, not defaulted;
  the reserved registry slot may be revisited by a later change but this one
  closes the question for `#Derive`'s first expansion
  ([design.md](design.md#derive-re-target)).
- **Two implable conversion traits.** Keeping `ToString` implantable beside
  `Display` re-creates the two-hooks confusion this change removes
  ([design.md](design.md#tostring-is-derived-never-written)).
- **A sink-shaped or macro-shaped contract.** `display` returns `string`; the
  language has no macro system for a `write!` equivalent
  ([design.md](design.md#method-shape-return-string-not-a-sink)).
- **Changing what interpolation means.** The acceptance set, the outputs,
  and the error code are preserved; only the implemented name changes.

## Readiness

- [x] Alternatives — contracts' home, hook split, method shape, `Format`
      naming, `Debug`'s parameter, derive re-target, field rules, `print`
      precedence — [design.md](design.md)
- [x] Replacement text for the eleven chapters —
      [specs/builtins/prelude.md](specs/builtins/prelude.md),
      [specs/builtins/README.md](specs/builtins/README.md),
      [specs/builtins/intrinsic-functions.md](specs/builtins/intrinsic-functions.md),
      [specs/types/traits.md](specs/types/traits.md),
      [specs/types/enums.md](specs/types/enums.md),
      [specs/types/composite-types.md](specs/types/composite-types.md),
      [specs/type-system/coercion.md](specs/type-system/coercion.md),
      [specs/type-system/checking-rules.md](specs/type-system/checking-rules.md),
      [specs/type-system/errors.md](specs/type-system/errors.md),
      [specs/expressions/literals.md](specs/expressions/literals.md),
      [specs/names.md](specs/names.md)
- [x] No grammar, lexical, or AST delta — identifiers only; the production
      count is unchanged
- [x] Reviewer cross-check of each delta against the chapter it replaces
      (the [workflow](../README.md#workflow) requires it before `review`)
- [ ] Cross-repository note: the standard library specification's formatting
      module cites these contracts once folded; its change belongs to that
      repository's index, not this one's

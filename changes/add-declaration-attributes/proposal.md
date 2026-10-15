# Add Declaration Attributes

**Status:** draft · **Date:** 2026-10-15 · **Affects:** `lexical-structure.md`,
`grammar.md`, `types/traits.md`, `ast.md`, `type-system/errors.md`

## Summary

A declaration MAY carry attributes written directly before it — `#Name`, or
`#Name` followed by a parenthesized list of trait names:

```xulo
#Derive(ToString)
struct User { name: String, age: Int }
```

The attribute *form* is generic; the set of names the checker interprets is a
**closed registry**. An attribute whose name is not in the registry is a
compile-time error, and there are no inert attributes — every attribute
written in source means something to the checker. The first registered
attribute is `#Derive(…)`, whose argument list names derive targets. The only
registered target is `ToString`: on a `struct` or an `enum`,
`#Derive(ToString)` generates exactly the structural `ToString` impl an author
would otherwise write by hand.

Until this change is accepted, `#` is a reserved symbol that MUST NOT appear
in source ([`../../lexical-structure.md`](../../lexical-structure.md)), no
attribute production exists
([`../../grammar.md`](../../grammar.md)), and every `ToString` impl is
written by hand
([`../../types/traits.md`](../../types/traits.md)). The deltas in
[`specs/`](specs/) are the text the chapters gain once the change is
accepted.

## Motivation

### The structural `ToString` impl is boilerplate

`ToString` is the hook that gives a user type a string form: `str()` and
template interpolation consult it, and a type without it cannot appear in
`` `${…}` `` (`E0212`). For the common case — a struct whose fields are all
base types or themselves stringable — the impl writes itself field by field,
and every codebase writes the same one:

```xulo
struct User { name: String, age: Int }

impl ToString for User {
  fn to_string(self): String {
    `${self.name} (${self.age})`
  }
}
```

`#Derive(ToString)` replaces that impl with one line and keeps the semantics
identical, so the author spends the impl on the *custom* shape — the
`User(name: …, age: …)` above is already what `print` gives a map — and
reaches for the hand-written form only when the default is not wanted.

### The structural rendering already exists as a language concept

The output rules of
[`../../builtins/intrinsic-functions.md`](../../builtins/intrinsic-functions.md)
render an enum value structurally — `Enum::Variant`, or `Enum::Variant(p, …)`
with payloads — with no author cooperation. Derive extends the same idea to
user types through the one trait that already serves as the string-form hook:
the generated impl renders a struct as `User(name: "Alice", age: 30)` and an
enum by the table's own variant form, field by field, recursively through
`ToString`. Nothing new is invented for the rendering; derive only automates
the impl that reaches it.

### The mechanism must be language-level

Xulo has no macros: attributes are interpreted by the checker, and nothing
else can register or expand them. A derive that generates an impl therefore
has to be a checker-known name, and the minimal honest form of
checker-known names is a closed registry with an error for everything else.
That is also what leaves the door open without opening a mechanism: `Display`
and `Debug` — the std/fmt traits a future change may register as further
targets — name no extension point today, only reserved registry slots.

## Scope

### In scope

- **Lexical change.** `#` leaves the reserved-symbols table and becomes a
  token of the inventory.
- **Syntax.** An attribute production — `#Name` with an optional
  parenthesized, comma-separated identifier list — and an attribute list on
  the five item declarations: `fn`, `struct`, `enum`, `trait`, and `impl`.
- **Registry rule.** Attributes form a closed, checker-interpreted set; an
  unregistered attribute name is a diagnostic; attribute names occupy their
  own namespace.
- **`#Derive`.** The registered attribute, its argument shape, its valid
  positions (`struct`, `enum`), the structural expansion for the target
  `ToString`, and the checks the expansion implies.
- **Diagnostics.** Five new codes in the names-and-declarations family
  (`E0108`–`E0112`), and the reuse of `E0805` and `E0801` where the
  expansion collides with an existing impl or a field has no string form.

### Out of scope

- **Registering `Display` or `Debug`.** Their definitions belong to std/fmt
  — outside this specification
  ([`../../README.md`](../../README.md), § 1.1) — and their consumers
  (template format specifiers for `Display`, a debug-rendering entry point
  for `Debug`) do not exist yet. The names are reserved in the derive
  registry and rejected with `E0111` until a future change registers them.
- **Template format specifiers.** `` `${x:.2}` `` is a template-grammar
  extension with its own design; this change touches templates not at all.
- **Attributes on bindings and tests.** `let`, `const`, `test`, type
  aliases, and `pub use` carry no attribute list in this version; extending
  the grammar to them is a language change whenever a registered attribute
  needs them.
- **Component declarations.** `@State`, `@Store`, `@Effect`, and
  `@Environment` are the component family and keep their own marker; `#`
  does not appear in component bodies and `@` does not appear on ordinary
  declarations ([design.md](design.md)).
- **Non-identifier arguments.** Attribute arguments are trait names, not
  expressions; a future attribute that needs a value argument extends the
  production then.
- **Derive on a generic type.** `#Derive(ToString)` on a type that declares
  type parameters is rejected (`E0112`); lifting the restriction needs a way
  to bound a parameter over "has a string form" that base types satisfy,
  which the language does not yet have
  ([design.md](design.md#derive-on-a-generic-type)).

## Compatibility

- **Purely additive.** Before this change `#` is a lexical error anywhere in
  source, so no well-formed program gains or loses a reading; the only new
  behavior is attached to syntax that did not previously exist. A program
  acquires a new diagnostic only by writing an attribute — an unknown name, a
  wrong position, a wrong argument shape, an unregistered target, a generic
  derive, or a derive that collides with a hand-written impl (`E0805`).
- **Hand-written impls are untouched.** The derive generates exactly the impl
  the author would have written; deleting the attribute and writing the impl
  by hand preserves the program's meaning.

## Non-goals

- **An open registry, macros, or a plugin mechanism** — the opposite of the
  design; attributes are checker-interpreted and nothing else
  ([design.md](design.md)).
- **Inert attributes** — an attribute the checker ignores would promise a
  tooling contract Xulo does not have.
- **Migrating the component family to `#`** — `@` stays where it is
  ([design.md](design.md#marker-territory--vs-)).
- **Derive targets beyond `ToString`** in this change; `Display` and `Debug`
  arrive, if they arrive, with their std/fmt definitions and consumers in a
  change of their own.

## Readiness

- [x] Alternatives, marker territory, registry rule, and expansion semantics —
      [design.md](design.md)
- [x] Grammar and lexical delta —
      [design.md](design.md#grammar-delta),
      [specs/grammar.md](specs/grammar.md),
      [specs/lexical-structure.md](specs/lexical-structure.md)
- [x] Replacement text for `types/traits.md` —
      [specs/traits.md](specs/traits.md)
- [x] AST delta — [specs/ast.md](specs/ast.md)
- [ ] Registration of `E0108`–`E0112` in
      [`../../type-system/errors.md`](../../type-system/errors.md), which
      [design.md](design.md#diagnostics) makes part of acceptance — the codes
      are named by the deltas but belong to no main chapter until the change
      is folded in.

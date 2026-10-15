# Design: Declaration Attributes

This document records the alternatives for declaration attributes, the design
chosen, and the consequences that choice carries: which marker owns which
territory, how the registry is closed, what `#Derive` generates, and what the
grammar and the diagnostics become. The prose that replaces the chapter text
is in [specs/](specs/); the motivation and the scope are in
[proposal.md](proposal.md).

## Marker territory — `@` vs `#`

### `#` for declarations, `@` left to the component family — RECOMMENDED

- **Pro:** each marker keeps the territory it already has. `@` introduced
  exactly four component forms — `@State`, `@Store`, `@Effect`,
  `@Environment` — governed by the `E07xx` family, and
  [`../../ast.md`](../../ast.md) already records that set as closed rather
  than a general attribute syntax. `#` was reserved with no meaning at all,
  so the split costs nothing and both families stay small and greppable:
  component state is `@`, declaration modifiers are `#`.
- **Con:** two prefix markers instead of one; a reader meets both.

**Decision.** `@` remains the component-family marker and gains no further
forms; `#` becomes the declaration-attribute marker. Neither grammar admits
the other's marker: `#` appears on no component declaration, `@` on no
ordinary declaration.

### Migrate the component family to `#` — `#state`, `#store`, …

- **Pro:** one marker for everything that prefixes a declaration.
- **Con:** renames a form the specification has shipped and the `E07xx`
  diagnostics speak about, for a gain that is only one token; and it would
  fold the closed component set into the general registry, where a future
  `#State` registered by nothing could be written. Rejected.

### Extend `@` to ordinary declarations — `@Derive(ToString)`

- **Pro:** one marker, no lexical change (`@` is already a token).
- **Con:** the component set stops being closed — `grammar.md` would admit
  `@` forms that are not components, and `ast.md`'s four dedicated nodes
  would need the `Attribute` wrapper the note explicitly declines. The
  component family's meaning ("this declaration is UI state") and the
  attribute's meaning ("modify how this declaration is processed") are
  different ideas sharing one sigil. Rejected.

## Attribute form and registry

### Generic form, closed registry — `#Name` with optional `(args)` — RECOMMENDED

- **Pro:** one production covers every present and future attribute; the
  closed registry is the smallest mechanism that can honestly exist without
  macros — the checker interprets every attribute it accepts, and a name it
  does not interpret is an error rather than a comment.
- **Con:** the registry is a compiler list; adding to it is a language change.

**Decision.** The grammar admits `#Name` on the five item declarations. The
semantics are:

> The attribute names the checker interprets form a closed registry. Each
> entry states the name, the declarations it is valid on, the shape of its
> argument list, and what it prescribes. An attribute whose name is not in
> the registry is `E0108`; a registered attribute written on a declaration
> the registry does not allow it on is `E0109`; a registered attribute whose
> arguments do not fit the shape the registry states is `E0110`.

Attribute names occupy their own namespace — the name after `#` binds
nothing, so `#Derive` does not take `Derive` away from user code and
introduces no `E0106`-style reservation.

### Inert attributes — rejected

An attribute the checker parses and ignores would be a lie in a language with
no macro system and no tooling contract: nothing downstream consumes it, and
the specification would be specifying a comment syntax with angle brackets.
Every attribute written in source is interpreted, or it is an error.

### An open registry — rejected

Letting the standard library or a package register attributes requires an
extension mechanism — macros, plugins, or an initialization protocol — all
of which are outside the language as specified. The registry grows by
changes to this specification, which is how every other closed set in Xulo
grows (intrinsics, the four `@` forms, the error codes themselves).

### Keyword form — `derive ToString { … }` — rejected

One keyword per attribute takes a name from the keyword pool and does not
scale past the first two attributes without a second marker eventually being
invented anyway. The prefix form is the generic one.

## Placement

**Decision.** An `AttributeList` precedes the declaration on the five item
forms — `FnDecl`, `StructDecl`, `EnumDecl`, `TraitDecl`, `ImplDecl` — outermost,
before `pub` and `async`:

```xulo
#Derive(ToString)
pub struct User { name: String, age: Int }

#Derive(ToString)
#Derive(ToString)          // error[E0805]: the impl would be generated twice
enum Shape { Circle(Float), Rect(Float, Float) }
```

The grammar is uniform and the registry decides validity, so extending the
set of attributes later touches no production — only the registry. Binding
forms (`let`, `const`), `test`, type aliases, and `pub use` carry no attribute
list in this version; the grammar gains it when a registered attribute needs
it. A local `fn` inside a block parses with the same `FnDecl` production and
so may carry the list syntactically; `#Derive` is then rejected by position
(`E0109`), exactly as it is on a top-level `fn`.

The alternative — admitting the list only on `struct` and `enum`, the two
positions `#Derive` accepts — was rejected as too narrow for a *generic*
attribute system: every future target (an `#Inline` on `fn`, say) would need
a grammar edit in addition to its registry entry.

## Arguments

**Decision.** Arguments are identifiers, comma-separated, with an optional
trailing comma — the shape of a trait-name list:

```text
AttributeArgs = AttributeArg { "," AttributeArg } [ "," ] ;
AttributeArg  = Identifier ;
```

`#Derive` takes one or more registered targets; `#Derive` with no argument
list and `#Derive()` with an empty one are both `E0110`: the target list is
one or more names, never zero. Expressions, strings, and numbers are not
argument forms: the one registered attribute needs only names, and a future
attribute that needs a value extends `AttributeArg` then.

## Derive

### Expansion

**Decision.** `#Derive(ToString)` on a `struct` or an `enum` generates
exactly the impl an author would write by hand for the structural case — the
body is fixed by the target, not chosen by the author:

On a struct `S` with fields `f₁ … fₙ`, the generated impl is:

```xulo
impl ToString for S {
  fn to_string(self): String {
    `S(f1: ${self.f1}, f2: ${self.f2})`     // n fields, in declaration order
  }
}
```

with `S()` for a struct with no fields, and with every `${…}` interpolated by
the ordinary rules — a base type renders as itself, a type implementing
`ToString` renders through its `to_string`. The rendering matches what the
output table of
[`../../builtins/intrinsic-functions.md`](../../builtins/intrinsic-functions.md)
already gives a map: `User(name: "Alice", age: 30)`.

On an enum `E`, the generated body matches on `self` and renders each variant
exactly as the output table renders an enum value — `E::V`, or `E::V(p, …)`
with every payload rendered by the field rules above:

```xulo
#Derive(ToString)
enum Shape { Circle(Float), Rect(Float, Float) }

// `Shape::Circle(2.5).to_string()` is `Shape::Circle(2.5)`
// `Shape::Rect(3.0, 4.0).to_string()` is `Shape::Rect(3.0, 4.0)`
```

Payload slots of a named payload are rendered positionally, construction and
matching being positional throughout
([`../../types/enums.md`](../../types/enums.md)).

### Checks the expansion implies

- **A field with no string form fails the derive.** A field (or payload)
  type that is neither a base type nor a `ToString` impl — a `List`, a `Map`,
  an optional, another derived type's non-stringable field — makes the
  generated interpolation ill-formed; it is reported as `E0801` with the
  field's type as the span: `` `List<Int>` does not implement `ToString` ``.
- **A manual impl and a derive collide.** The expansion generates an impl, so
  a hand-written `impl ToString` for the same type — or a second
  `#Derive(ToString)` — is the ordinary conflicting-impl error `E0805`.
- **No recursion hazard.** A cycle would require a stringable field that
  contains the type itself; every container that could hold one (`List`,
  `Map`, `Set`, optionals of the type) has no string form of its own, so the
  field check rejects the cycle before the body can recurse.

### Derive on a generic type

**Decision.** `#Derive(ToString)` on a type that declares type parameters is
rejected (`E0112`): ``cannot derive `ToString` for a generic type``.

The obstacle is expressibility, not implementation. A derived impl for
`Box<T> { v: T }` would have to assume `T` has a string form, but the only
bound the language can write is `T: ToString` — and that bound excludes the
instantiations `str` already accepts, where `T` is a base type that has a
string form without implementing anything. A correct bound ("`T` is a base
type or implements `ToString`") does not exist, and inventing it — or giving
base types `ToString` impls — is a generics change of its own. Until then
the hand-written impl, which can be written per-instantiation or bounded by
hand, remains the route for a generic type.

### What derive does not do

- It does not affect `print`. The output table of
  [`../../builtins/intrinsic-functions.md`](../../builtins/intrinsic-functions.md)
  renders base types, containers, and enum values, and does not consult
  `ToString` (whether `print` of a `ToString` type should is an adjacent gap
  this change does not close). The consumers of the derived impl are the ones
  `ToString` already has: `str()` and template interpolation.
- It does not register names. `#Derive` binds nothing; the generated impl is
  an ordinary impl, visible to dispatch exactly as a written one.

## Registered targets — `ToString`, and the reservation of `Display` and `Debug`

**Decision.** The derive-target registry is closed in the same way the
attribute registry is. Version one registers exactly one target, `ToString`,
whose trait is built into the prelude
([`../../types/traits.md`](../../types/traits.md)). `Display` and `Debug`
are **reserved names in this registry**: written as targets they are `E0111`,
and nothing else in the language is required to know them.

The reservation records an intent without specifying anything: when std/fmt
defines those traits — their definitions live outside this specification
([`../../README.md`](../../README.md), § 1.1) and their consumers are
template format specifiers for `Display` and a debug-rendering entry point
for `Debug` — a change registers them as targets, carrying whatever minimum
signature surface the expansion needs. The constraint that change inherits is
recorded here: because Xulo has no macros, a derivable trait is necessarily
one whose name the checker knows, so registering a std/fmt trait as a target
is a language-level act even though the trait's definition is a library one.

## Grammar delta

`#` joins `@` and `$` in the symbol inventory, and the declarations section
gains the attribute productions, attached to the five item forms:

```text
SymbolOperator =
      "." | "?" | "!" | "=" | "==" | "!=" | "<" | ">" | "<=" | ">="
    | "+" | "-" | "*" | "/" | "%" | "**" | "<<" | ">>" | "&" | "|" | "^" | "~"
    | "=>" | "..<" | "..." | "?." | "??" | "@" | "$" | "::" | ":=" | "#" ;

Attribute     = "#" Identifier [ "(" [ AttributeArgs ] ")" ] ;
AttributeArgs = AttributeArg { "," AttributeArg } [ "," ] ;
AttributeArg  = Identifier ;
AttributeList = { Attribute } ;

FnDecl    = [ AttributeList ] [ "pub" ] [ "async" ] "fn" Identifier
            GenericParams? "(" [ ParameterList ] ")" [ ":" Type ]
            WhereClause? Block ;
StructDecl = [ AttributeList ] [ "pub" ] "struct" Identifier GenericParams?
             "{" [ FieldList ] "}" ;
EnumDecl   = [ AttributeList ] [ "pub" ] "enum" Identifier GenericParams?
             "{" [ VariantList ] "}" ;
TraitDecl  = [ AttributeList ] [ "pub" ] "trait" Identifier GenericParams?
             "{" [ TraitMethods ] "}" ;
ImplDecl   = [ AttributeList ] "impl" GenericParams? [ Identifier "for" ]
             Type "{" { FnDecl } "}" ;
```

**Ambiguity note.** None arises: `#` began no token before this change, so
every position that now lexes `#` was previously a lexical error. Within a
template, `#` is ordinary template text and starts no attribute — attributes
are written between declarations, where no template is open.

## Diagnostics

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0108` | an attribute name that is in no registry | `#Inline` | `` attribute `Inline` is not defined `` |
| `E0109` | a registered attribute on a declaration the registry does not allow it on | `#Derive(ToString) fn main()` | `` attribute `Derive` is not valid on a function `` |
| `E0110` | a registered attribute whose arguments do not fit the registered shape — no list where one is required, or an empty one where one or more arguments are | `#Derive`, `#Derive()` | `` attribute `Derive` takes a parenthesized list of trait names `` |
| `E0111` | a well-shaped argument that names a reserved, unregistered derive target | `#Derive(Display)` | `` `Display` is not a registered derive target `` |
| `E0112` | `#Derive` on a type that declares type parameters | `#Derive(ToString) struct Box<T> { v: T }` | `` cannot derive `ToString` for a generic type `` |
| `E0805` | the expansion collides with an existing impl — a hand-written `impl ToString`, or a second `#Derive(ToString)` | a manual impl beside the attribute | `` `ToString` is already implemented for `User` `` |
| `E0801` | a field or payload type with no string form | `#Derive(ToString) struct W { xs: List<Int> }` | `` `List<Int>` does not implement `ToString` `` |

`E0111` carries the note `= note: registered targets: `ToString``. The five
new codes join the names-and-declarations family `E0101`–`E0199`, whose
highest assigned code today is `E0107`; `E0805` and `E0801` are reused with
their existing meanings — the expansion is an ordinary impl, and its
interpolations are ordinary template positions.

## Alternatives rejected

- **Inert and tooling attributes** — no macros, no consumers; a comment with
  a sigil.
- **An open registry** — an extension mechanism the language does not have.
- **Expression arguments** — the one registered attribute needs names only.
- **One keyword per attribute** — keyword pool churn, no generic form.
- **Folding the component family into the registry** — `@` keeps its closed
  set; the two families are different ideas.
- **Derive that emits a body the author customizes** (a payload block, a
  formatting string) — the structural case is fixed; the custom case is the
  hand-written impl, and a customizable derive is a macro wearing a hat.

## Impact

The chapters named in [proposal.md](proposal.md), and what each gains:

| Chapter | Change |
|---------|--------|
| [`../../lexical-structure.md`](../../lexical-structure.md) | `#` leaves the reserved-symbols table, joins the token inventory, and the conformance list of lexical errors loses its `#` clause; see [specs/lexical-structure.md](specs/lexical-structure.md) |
| [`../../grammar.md`](../../grammar.md) | `#` in `SymbolOperator`, the `Attribute` productions, the attribute list on the five item forms, and the index; see [specs/grammar.md](specs/grammar.md) |
| [`../../types/traits.md`](../../types/traits.md) | a new **Attributes** section — registry rule, `#Derive`, the expansion and its checks; see [specs/traits.md](specs/traits.md) |
| [`../../ast.md`](../../ast.md) | `AttributeNode`, an `attributes` field on the five declaration nodes, and the component-family note; see [specs/ast.md](specs/ast.md) |
| [`../../type-system/errors.md`](../../type-system/errors.md) | `E0108`–`E0112` in the names-and-declarations table, and a footnote reserving `Display` and `Debug`; see [specs/errors.md](specs/errors.md) |

The change touches no part of `modules/`, `components/`, the type-checker
chapters, or the expression layer: imports and visibility, component bodies
and `$` binding, inference and bounds, and every expression production are
unaffected. A declaration that carries no attribute parses and checks exactly
as it does today.

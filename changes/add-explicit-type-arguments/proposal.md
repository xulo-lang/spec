# Add Explicit Type Arguments

**Status:** draft · **Date:** 2026-10-06 · **Affects:** `types/generics.md`,
`types/README.md`, `functions.md`, `expressions/calls.md`,
`type-system/checking-rules.md`, `grammar.md`

## Summary

A call MAY carry an explicit type-argument list between the callee and the
argument list — `first<int>(…)`, `map<string, int>(…)`. The written arguments
replace inference for that call: they are checked against the callee's
declaration for arity and bounds, and then used directly in place of solving.
A call that omits the list is checked exactly as it is today.

Until this change is accepted, the chapters named above remain normative and
`first<int>(…)` stays ill-formed: [`../../types/generics.md`](../../types/generics.md)
states that the language has no syntax for explicit type arguments in a call,
[`../../expressions/calls.md`](../../expressions/calls.md) states that
writing one is not well-formed, and [`../../functions.md`](../../functions.md)
and [`../../types/README.md`](../../types/README.md) repeat the same rule.
The deltas in
[`specs/generics.md`](specs/generics.md) are the text that replaces those
passages once the change is accepted.

## Motivation

### The result type is not determined by the arguments

A generic function may fix its type parameter only through its return type,
so nothing about the value arguments constrains it. Today the call is
repaired only by arranging an expected type, and the only place to put one is
the surrounding context:

```xulo
fn empty<T>(): list<T> { [] }

let xs = empty()             // error[E0216]: T is not determined
let ys: list<int> = empty()   // accepted: the annotation supplies T
let zs = empty<int>()         // accepted: the call supplies T
```

The annotation must be attached to a construct that can carry one, so a call
in a position with no annotation to write still fails, and the repair is a
temporary binding whose sole purpose is to hold the type:

```xulo
let head = empty()[0]          // error[E0216]: no expected type reaches `empty`
```

```xulo
let tmp: list<int> = empty()   // a binding whose only purpose is to carry the type
let head = tmp[0]
```

```xulo
let head = empty<int>()[0]     // the type is written where it is needed
```

The same failure occurs in the prelude, where the entries of a map are the
only place its key and value types could come from:

```xulo
let annotated: map<string, int> = map_from_entries([])  // today: K and V come from the annotation
let written = map_from_entries<string, int>([])         // proposed: K and V are written at the call
```

### The type is pinned where the call is read

An annotation pins the type on the binding, at a distance from the call whose
instantiation it constrains, and that distance grows with the size of the
expression. Writing the type arguments on the call itself keeps the two
together, so the intended instantiation is visible in the line that performs
the call:

```xulo
let by_annotation: map<string, int> = map_from_entries([])  // pinned away from the call
let by_call = map_from_entries<string, int>([])             // pinned on the call
```

It also records the author's choice rather than a property the checker
derived: a later edit to an argument or to a parameter type cannot silently
retarget the instantiation of a call whose type arguments were written, and a
reader — or a tool that reports the type of an expression — sees what the
source says instead of what inference happened to conclude.

### One failure becomes a run of diagnostics

An unsolved parameter is reported at its call, and every expression that
consumes the resulting binding is then reported in turn, since an
implementation MAY report further failures after the first. The root cause is
one missing type argument, but it is charged many times over:

```xulo
fn empty<T>(): list<T> { [] }

let xs = empty()        // error[E0216]: T is not determined
let ys = xs + [1]       // reported as well: `xs` never received a type
let n = xs[0] + 1       // and again here
```

Writing `empty<int>()` repairs the root cause at the call, and the
downstream diagnostics disappear with it.

## Scope

### In scope

- **Syntax.** The call production gains an optional type-argument list, with
  the disambiguation rule against relational `<` ([design.md](design.md)).
- **Typing.** The rules for arity, bounds, substitution of the written
  arguments, all-or-nothing specification, and the suppression of inference
  for that call.
- **Diagnostics.** The codes raised when a type-argument list conflicts with
  the declaration it names, and the note offered when `<` was intended as a
  comparison.

### Out of scope

- **Explicit type arguments on `struct` and `enum` constructors.** RECOMMENDED
  OUT for version one. Constructors are written `Pair(key: …, value: …)` where
  the name is a type rather than a value, their type arguments are determined
  by their field initializers, and no call has yet failed for want of a
  written list; since the parse rule claims the token shape, a list written
  before a constructor is diagnosed with `E0807` rather than read as a
  comparison.
- **Turbofish.** The turbofish spelling — a `::` token between the callee and
  the `<` — is rejected: `::` is reserved for enum-variant paths
  (`Theme::Dark`, `Shape::Circle(2)`) in expressions and in patterns alike,
  and a second use of it in expression position would weaken that rule.
- **Partial specification.** A call writes all of the callee's type arguments
  or none of them; writing a subset that lets the remainder be inferred is not
  part of the language.

## Compatibility

- **Purely additive.** Type arguments are written for a call or not at all,
  and inference is unchanged where the list is omitted. A program gains a new
  diagnostic only if it writes the new syntax against a declaration that will
  not take it — wrong arity, unsatisfied bound, arguments that no longer fit,
  or a list before a callee that accepts none.
- **One shape parses differently.** The only expressions whose reading changes
  are those of the shape `ident < … > ( … )`, whose two readings are separated
  by the rule in [design.md](design.md). The common form `a < b > (c)` is
  already rejected by the non-associative relational rules, so no well-formed
  comparison is lost, and the one corner in which both readings are
  well-formed — `f(a < b, c > (d))` — is repaired by parenthesizing a
  comparison: `f((a < b), c > (d))`.

## Non-goals

- **Default type parameters** (`fn empty<T = int>()`): a separate feature with
  its own interaction with bounds and with inference.
- **Variadic generics**: a type-parameter list whose length is not fixed by
  the declaration.
- **Explicit type arguments in patterns**: patterns deconstruct values, and
  no pattern position names a generic callee.

## Readiness

- [x] Alternatives, chosen design, and rationale — [design.md](design.md)
- [x] Disambiguation rule for `<` and the grammar delta —
      [design.md](design.md#grammar-delta)
- [x] Typing rule, inference interaction, and diagnostics —
      [design.md](design.md#typing-rule)
- [x] Replacement text for `types/generics.md` —
      [specs/generics.md](specs/generics.md)
- [ ] Deltas for the remaining affected chapters — `expressions/calls.md`,
      `type-system/checking-rules.md`, `functions.md`, `types/README.md`, and
      `grammar.md` — are added to `specs/` before this change leaves `draft`,
      together with the registration of the two new diagnostic codes in
      [`../../type-system/errors.md`](../../type-system/errors.md).

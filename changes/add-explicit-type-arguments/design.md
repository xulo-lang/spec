# Design: Explicit Type Arguments

This document records the alternatives for the syntax of explicit type
arguments, the design chosen, and the consequences that choice carries: how
`<` is told apart from the relational operator, how written type arguments are
checked, how they interact with inference, and what the grammar and the
diagnostics become. The prose that replaces the chapter text is in
[specs/generics.md](specs/generics.md); the motivation and the scope are in
[proposal.md](proposal.md).

## Syntax alternatives considered

### Angle brackets after the callee — `first<int>(…)` — RECOMMENDED

- **Pro:** reads exactly like the type-parameter list of the declaration and
  like every other angle-bracket type in the language; no new token, no new
  keyword.
- **Con:** `<` immediately follows an identifier in call position, which is
  also the token sequence of a comparison.

**The parse problem.** The sequence `IDENT < … > ( … )` has two readings. Read
as an expression, `x = a < b > (c)` is `(a < b) > (c)` — and since the
relational operators are non-associative, that reading does not parse in the
grammar of [`../../expressions/operators.md`](../../expressions/operators.md)
for the same reason `a < b < c` does not. The type-argument reading,
`a<b>(c)`, therefore takes no well-formed program away from that shape. The
one shape in which both readings are well-formed puts a top-level comma inside
the material, where the comparison reading takes it for an argument separator:
`f(a < b, c > (d))` reads today as two comparisons, one per argument.

**Decision.** Adopt the angle-bracket form with this disambiguation rule:

> After an identifier in call position, `<` starts a type-argument list only
> if both conditions hold:
>
> 1. the material between that `<` and the matching `>` parses as a
>    comma-separated list of types — the empty material counts as a
>    zero-length list, so `f<>(…)` reaches the arity diagnostic instead of
>    falling back to a comparison; and
> 2. the token immediately after the matching `>` is `(`.
>
> If either condition fails, the `<` is the relational operator and the
> expression is a comparison. The scan for the matching `>` counts angle depth
> and skips over `(` `)`, `[` `]`, and `{` `}` groups. The decision is purely
> syntactic: name resolution MUST NOT be consulted, and backtracking from the
> type-argument reading to the comparison reading is allowed whenever the
> call parse does not complete.

Under the rule, `x = a < b > (c)` is read as the call `a<b>(c)`, and
`f(a < b, c > (d))` is read as `f(a<b, c>(d))`; an expression that means two
comparisons is written with one of them parenthesized — `f((a < b), c > (d))`
— where the material after `<` no longer parses as a type list. Two further
consequences of the token inventory:

- `>>` and `>=` are single tokens and never match, so a nested type argument
  keeps the spacing the nested-bracket rule already requires:
  `first<Map<string, int> >(m)`. Written without the space, the scan finds no
  matching `>` and the `<` is read relationally, which fails to parse and is
  diagnosed with a note to insert the space.
- The rule is scoped to an identifier in call position. `<` in a type
  position, in a declaration's parameter list, and in a bound is already
  unambiguous, and every generic callee ends in an identifier — a call through
  a value of function type has no type parameters and accepts no list.

### Turbofish — `first::<int>(…)`

- **Pro:** unambiguous with no lookahead: the `<` follows `::` rather than an
  identifier, so it cannot be mistaken for the relational operator.
- **Con:** overloads `::`, which is the variant path separator in expressions
  and in patterns alike
  ([`../../types/enums.md`](../../types/enums.md)); allowing it after an
  ordinary value identifier would give a marker reserved for paths a second,
  unrelated meaning. Rejected.

### Keyword form — `first explicit <int>(…)`

- **Pro:** the marker is impossible to confuse with a comparison.
- **Con:** verbose at every use; the marker would have to be reserved as a
  keyword, taking an ordinary identifier away from user code; and a token that
  appears nowhere else in the expression grammar has to be learned for this
  one construct. Rejected.

### Bracket form — `first[int](…)`

- **Pro:** `[` is unambiguous in call position.
- **Con:** collides with the subscript, which is a postfix operator: `first`
  is a value, `first[int]` is a subscript of it, and the reading would be an
  index followed by a call. The brackets are taken. Rejected.

## Typing rule

Let the callee's declaration carry type parameters `T₁ … Tₙ` with their
bounds. A call that writes a type-argument list `A₁ … Aₘ` is checked by these
rules, in order:

1. **Arity.** `m` MUST equal `n`; a longer or shorter list is `E0808`. The
   code is free: the family `E0801`–`E0899` is the trait and generic range of
   [`../../type-system/errors.md`](../../type-system/errors.md), which
   assigns `E0801`–`E0806` and no higher code.
2. **All or nothing.** Every type parameter is written or none is. A partial
   list is not a distinct form that leaves the rest to inference; it is the
   arity failure of rule 1.
3. **Bounds.** Each `Aᵢ` MUST satisfy every bound of `Tᵢ`, reported as
   `E0801`, exactly as an inferred argument must be.
4. **Substitution without inference.** For each `i` the parameter is bound
   `Tᵢ := Aᵢ` before any constraint is generated: no type variable is
   introduced for it, no unification constraint is raised against the value
   arguments, and no expected type may override it. The value arguments are
   then checked against the substituted parameter types, and a mismatch is the
   ordinary argument type error `E0202`.
5. **Result.** The type of the call is the declared return type with every
   `Tᵢ := Aᵢ` substituted.
6. **Callee that accepts no list.** Explicit type arguments on a non-generic
   callee are `E0807`; the code is free for the same reason as `E0808`. The
   rule covers a value of function type as well: `f<int>(…)` where
   `f: fn(int): int` is `E0807`, not an empty instantiation. A `struct` or
   `enum` constructor accepts no list either — its type arguments are always
   determined by its field initializers — and is reported with `E0807`, so
   that the token shape is diagnosed rather than read as a comparison.

Rules 1, 2, and 6 apply wherever the callee ends in an identifier — a method
(`r.first<int>(xs)`), a trait static (`Trait.method<int>(x)`), or a namespace
member (`Task.all<int>(…)`) — exactly as they apply to an ordinary function
name.

## Inference interaction

Explicit type arguments suppress inference entirely for that call: the
parameters named in the list are solved before constraint generation and take
part in no unification. Inference is not partially suspended — there is no
mixing of written and inferred parameters on one call.

Return-type-driven checking still applies to the result. The call has a closed
type as soon as its type arguments are written, and an enclosing expected type
checks that type rather than feeding anything back into the arguments:

```xulo
fn first<T>(xs: list<T>): T { xs[0] }

let n: int = first([1, 2, 3])              // inferred, unchanged
let m = first<int>([1, 2, 3])              // T = int, written
let k = first<int>(["a", "b"])             // error[E0202]: expected `list<int>`, found `list<string>`
let bad: string = first<int>([1, 2, 3])    // error[E0201]: expected `string`, found `int`
```

The expected type may still supply what the *value arguments* leave open when
the list is omitted, and it still checks the result when the list is written;
in either case it acts on types, and never overwrites a written argument.

```xulo
fn describe<T: Area>(shape: T): float { Area.area(shape) + Area.perimeter(shape) }

// `Rectangle` implements `Area`; `Square` does not
let a = describe<Rectangle>(r)     // T = Rectangle, checked against `T: Area`
let b = describe<Square>(s)        // error[E0801]: `Square` has no `Area` impl
```

## Grammar delta

The call production gains an optional type-argument list between the callee
and the argument list:

```text
Call         = Callee [ TypeArgs ] '(' [ ArgumentList ] ')' ;
TypeArgs     = '<' [ TypeList ] '>' ;
TypeList     = Type ( ',' Type )* ','? ;
```

**Ambiguity note.** `TypeArgs` is recognized only when `<` immediately
follows an identifier in call position and the disambiguation rule above
holds; everywhere else `<` is the relational operator. The spacing rule for
adjacent closing angle brackets applies to `TypeArgs` as it applies to every
other generic argument list, so `first<Map<string, int> >(m)` is well-formed
and `first<Map<string, int>>(m)` is not. Component calls, method calls, and
calls through a generic namespace member share the production.

## Diagnostics

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0807` | a type-argument list on a callee that accepts none — a non-generic function, a value of function type, a constructor call | `greet<int>("x")` for a non-generic `greet` | `` `greet` takes no type arguments `` |
| `E0808` | the list length differs from the declared type-parameter count | `first<int, string>(xs)` for `fn first<T>` | ``expected 1 type argument, found 2`` |
| `E0801` | an explicit type argument satisfies no bound | `describe<Square>(s)` | `` `Square` does not implement `Area` `` |
| `E0202` | an argument does not fit the substituted parameter type | `first<int>(["a"])` | ``expected `list<int>`, found `list<string>` `` |
| `E0201` | the call's instantiated result type disagrees with its context | `let bad: string = first<int>(…)` | ``expected `string`, found `int` `` |

Two failures belong to the parse, which carries no code of its own in the
diagnostic model:

- **Ambiguous `<`, comparison intended.** `x = a < b > (c)` gains the
  type-argument reading; the checker reports the call as ill-formed — `E0807`
  when the callee is a declared non-generic function, `E0204` when the callee
  has no function type at all — and attaches
  `= note: parenthesize the comparison: (a < b) > (c)`.
- **Material that is not a type list.** `first<Map<string, int>>(m)` closes
  with a `>>` token, so no matching `>` is found, the `<` is read relationally,
  and the expression fails to parse in the `syntax` category, with
  `= note: separate the closing brackets: first<Map<string, int> >(m)`.

Registration: on acceptance, `E0807` and `E0808` are added to the trait and
generic errors table of [`../../type-system/errors.md`](../../type-system/errors.md),
whose range `E0801`–`E0899` already covers them.

## Alternatives rejected

- **Explicit arguments as hints** that inference may confirm or override — two
  sources of truth for one parameter, with no way to diagnose a disagreement
  between them.
- **Partial specification** — a hybrid solver whose result depends on which
  parameters the author chose to pin, and on their order in the list.
- **Extending the same change to `struct` and `enum` constructors** — deferred:
  constructors infer from their field initializers, and no call has yet failed
  for want of a written list.

## Impact

The chapters named in [proposal.md](proposal.md), and what each gains:

| Chapter | Change |
|---------|--------|
| [`../../types/generics.md`](../../types/generics.md) | the inference rules gain the optional written list; see [specs/generics.md](specs/generics.md) |
| [`../../expressions/calls.md`](../../expressions/calls.md) | the `Call` production and the generic-call paragraph |
| [`../../type-system/checking-rules.md`](../../type-system/checking-rules.md) | the call and bounds rules gain arity, substitution, and the two new codes |
| `grammar.md` | the `Call` production and the ambiguity note |

The change touches no part of `modules/`, `components/`, or the async
expression layer: imports and visibility, component invocation and `$`
binding, and `Task` signatures are all unaffected. A call that omits the list
is untouched; the only expressions that parse differently are the
`ident < … > ( … )` shapes of the syntax section.

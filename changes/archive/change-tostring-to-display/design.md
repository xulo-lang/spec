# Design: String Form — Display, Debug, and Format

## Where the formatting contracts live

### In the language, as prelude contracts — RECOMMENDED

**Decision.** `Display`, `Debug`, and `Format` are defined by this
specification, in the prelude and the traits chapter, exactly where `ToString`
lives today.

Three reasons, in order of weight:

1. **The consumers are language constructs.** `${…}`, `str`, the `str`
   domain of `List.join`, and `#Derive` must name their protocol from inside
   type rules. A rule in this document cannot cite a protocol another
   document defines without making the citation run downhill: `E0212`,
   the coercion row, and the derive rules would all depend on a document
   whose own hierarchy (its README, § 6) says the opposite — that document
   cites *this* one for types and diagnostics, with `Result` and `Error` as
   the standing precedent. One family of citations exists; this design keeps
   it the only one.
2. **The std root's reachability does not change the document boundary.**
   The mechanical objection to a library-owned protocol — that `${}` cannot
   require an import — does not actually bite *on the other side's own
   terms*: the standard library's specification makes its `std` root
   compiler-provided and its qualified paths import-free (its modules
   chapter), so `std::fmt::Display` would resolve **there**. This
   specification contains no `std` root at all — § 1.1 holds `std/*` outside
   it — which is exactly why the boundary that bites is editorial: the
   language's error table would quote a library chapter's vocabulary
   (`precision`, `alternate`) and the two specifications would have to move
   in lockstep on one signature.
3. **The Rust shape says the same thing.** Rust's `Display` is documented
   under `std::fmt`, but its definition lives in `core::fmt` and `std` merely
   re-exports it — the part Rust put *below* allocation, because `no_std`
   must still format. Xulo has no allocation-free tier, so the technical
   reason is absent, but the *shape* it produced is the one chosen here —
   contracts below, engine above — with the layers of a different kind:
   Rust's are runtime dependency layers (core under alloc), xulo's are
   document layers (this specification under the other). The shape is what
   travels. The standard library's formatting
   module becomes a consumer of `Display`/`Debug` the way `std` members are
   already consumers of `Result`.

### Defined by the standard library, cited by the language — rejected

Mechanically workable (see above; Swift does exactly this with a stdlib
protocol that string interpolation consults). Rejected because it inverts
the citation direction the two documents already record — the standard
library's specification cites this one for types and diagnostics — makes
`E0212`'s message quote another spec's chapter, and moves a signature this
specification must freeze anyway into the other repository's dependency
list.

## The hook split

### `Display` carrying `Format`, `ToString` derived — RECOMMENDED

**Decision.** One implantable protocol (`Display`) whose method receives the
`Format`; the `toString` action is specified once — `v.toString()` is
`v.display` at the default `Format` — and no `impl` writes it.

The split only earns its names if the context-bearing side really carries
context: `Display` is *how a type is shown* (a `Temperature` at
`precision: 1` and at `precision: 0` is two texts), `toString` is *the
action* of getting a string (one answer, which is why it can be derived and
never written). If `Display` were also context-free, the two would be one
method under two names and the change would be ceremony.

The derivation is written as a language rule rather than a blanket impl
because Xulo has no blanket-impl mechanism — module-scoped `impl`s are
explicit and never inferred. Specifying the derived action beside `#Derive`'s
expansion is the same kind of checker-level stipulation the language already
makes.

### One context-bearing `ToString` — rejected

`fn toString(self, format: Format): string` would do the work but tells a
lie at every call site: `v.toString(format)` reads as "convert, given this
format", and the name says result where the parameter says manner. The
caller-facing vocabulary stays `toString` precisely because it *has* no
context — the derived, default-context action.

### Two implantable traits (`Display` + `ToString`) — rejected

This is the status quo's shape inverted: two hooks, two impls, and the
question every language with both (`Display`/`ToString` in Rust,
`__str__`/`str(x)` conventions elsewhere) spends documentation on — *which
one do I implement?* Here `ToString` is not implementable at all (an
`impl Display` that declares `fn toString` is `E0804`: the trait declares no
such method), so the question cannot be asked.

## Method shape: return string, not a sink

### `fn display(self, format: Format): string` — RECOMMENDED

**Decision.** The implementation returns text; consumers receive a value
they can post-process.

- **Xulo has no macro system** ([`introduction.md`](../../introduction.md),
  § "No macro system in v1"), so Rust's `write!(f, …)` — the thing that makes
  a sink ergonomic — cannot exist. The body of a `display` is a template
  literal, `` `(${self.x}, ${self.y})` ``, which is the same cognitive shape
  as Rust's inner `write!` but produces the string directly. The equivalent
  sink body would be a sequence of `f.write…` calls or a template built and
  then copied into the sink: double work with no reader benefit.
- **Layout is then guaranteed.** The consumer pads the *returned* string —
  width, alignment, and fill always apply, with no cooperation from the
  implementation. This removes the wart Rust documents for its sink shape:
  "alignment might not be implemented by some types", because a sink
  impl that ignores `f` has already written past the padding opportunity.
  With a returned value there is nothing to ignore: `Format` carries no
  layout fields in the first place (see [the field set](#format-carries-only-semantics)).
- **Nothing can fail.** Producing a `string` is total; `fmt::Result` exists
  in Rust only so a formatting impl can propagate a *stream's* failure, and
  the standard library's formatting module states it returns no `Result` at
  all. There is no error channel to declare.

### A sink parameter (`f: mut …`) — rejected

Xulo's borrowing (`p: mut T`, `mut self`) makes the shape writable, so the
objection is not syntax. It is that without a write-macro the sink buys
nothing: the caller must hold a `mut` place for the buffer, every impl
either appends piecewise (verbose, and piecewise appending of a *rendered
value* means formatting it into a string first anyway) or builds one string
and copies it in. The contract would also need a written failure story for a
`Result` that nothing in the language can produce here.

## `Format` carries only semantics

**Decision.** `Format` is `{ precision: int?, alternate: bool }`. Width,
alignment, fill, and zero-padding are **not** fields: they are layout, and
layout belongs to whoever asked for the rendering — the consumer applies it
to the returned string. The contract statement in `types/traits.md` says so
plainly, so an implementation never needs a padding helper and can never
half-honor one.

A `display` that has a meaning for `precision` (a `Money` rounding to
decimal places) or `alternate` (a hex-flavored rendering) reflects them and
MAY ignore them; ignoring them leaves the consumer's post-applied layout
intact either way.

This is the split the whole change exists to enable: **semantics in the
`Format`, layout in the engine, structure in `Debug`.**

### Everything a Rust `Formatter` exposes — rejected

Carrying `width`/`fill` into the contract would re-import the cooperation
problem (does the impl pad, or does the engine — double padding if both) and
would promise helper APIs (`pad_integral`-shaped) the language has no reason
to specify. The engine's own grammar (another document) still *has* width;
it simply never crosses this boundary.

## The context type is open

**Decision.** `Format` is an ordinary struct: anyone may construct
`Format(precision: 2, alternate: false)` and call `v.display(fmt)` directly.

Direct calls are a feature — asking a value for its form at a chosen
precision without routing through an engine — and construction is harmless:
the contract fixes what the fields *mean*, not who may fill them. Consumers
still construct `Format` per their own grammar; that is a rule about
consumers, not a lock on the type.

### A sealed type (constructible only by consumers) — rejected

Sealing needs an opaque type with no public constructor (`View`-shaped),
which then forbids the direct call above, and xulo structs have no private
constructors to reach the middle ground. Rust's `Formatter` is unconstructable
because it is tied to a sink's lifetime plumbing, not because openness is
dangerous; the plumbing is absent here.

### Extension policy — growth by optional-typed fields only

**Decision.** The field set may grow, and every added field MUST be of an
optional type.

The mechanism is the language's own — xulo has no field-default syntax, but
a field of type `T?` MAY be omitted at construction and takes `null` (the
initializer rule of
[`composite-types.md`](../../types/composite-types.md)). Openness makes a
field addition the one real risk of the open design: a *non-optional* added
field would fail every written `Format(…)` with `E0205` — a source break at
every direct call. The policy excludes exactly that shape. An optional added
field keeps every existing construction compiling and meaning what it meant:
omitted reads as "the consumer asked for nothing", which the contract
defines as the default rendering, and the default-`Format` definition grows
by the same `null`. A future *boolean* field is therefore `bool?` as well:
its `null` and its `false` share one declared reading — *not asked* — so the
third state costs nothing, while a non-optional `bool` is excluded not by
taste but by `E0205`. The two present fields are the policy's baseline:
`precision: int?` because `0` is a real request and only `null` can mean
*unasked*; `alternate: bool` because `false` already does. The policy is
written into the contract itself
([`specs/types/traits.md`](specs/types/traits.md)) so an implementer of a
future engine meets it beside the fields, not in this document.

## The context type's name

### `Format` — RECOMMENDED

**Decision.** The type is named `Format`, its parameter spelled as the
implementer likes (`format: Format`, `f: Format`).

- It is the user's concept, not the grammar's: "display me at this format"
  reads as intent, in the signature and in `format.precision ?? 1`.
- The field names keep Rust's getter vocabulary — `precision()`,
  `alternate()` are exactly what a Rust reader maps `format.precision` /
  `format.alternate` onto. The familiarity bridge was always going to be the
  member names; the type name's job is to not break the scene.
- It pairs with `Debug`: a future context-carrying `Debug` (an alternate
  pretty form) takes the same `Format`, so one name serves both contracts.
- Examples standardize on `format: Format`, and the choice is not cosmetic:
  a named argument spells the parameter's name at the call site
  ([`expressions/calls.md`](../../expressions/calls.md)), so the name is
  read twice — in the signature and in `display(format: …)` — and
  `names.md` asks for the descriptive over the abbreviated. `f: Format` and
  `fmt: Format` remain well-formed; the recommendation binds the
  documentation, not the language.

### `FormatSpec` — rejected

It is the nonterminal name of the format grammar in Rust's reference
(`format_spec := …`) — compiler-internal vocabulary. In user code it reads
like a stage direction from the parser leaking into the signature; that is
precisely the "出戏" the rename answers. A type may *have* a grammar; it
should not *be* one.

### `Formatter` — rejected

The `-er` agent noun promises an actor that formats into something, which is
Rust's sink object; a passive record of requested options under that name
misleads the readers who know Rust best (they arrive expecting `&mut`,
`write!`, and `f.pad()`, and the `: string` return has to un-teach all three).
It would also be the first agent-noun type in a vocabulary of plain nouns
(`Error`, `Range`, `Task`, `View`), and it invites the follow-up question —
why has `Formatter` no methods? — turning the field-vs-getter decision into
a second one.

### Bare `Spec` — rejected

Unqualified it names nothing (`spec` of what), and in this project's own
vocabulary "spec" already means the specification documents. A prelude name
should not squat on the project's most-used common noun.

## `Debug`'s signature carries `Format` too

### `fn debug(self, format: Format): string` — RECOMMENDED

**Decision.** Both contracts take the parameter from day one.

An impl must match the trait's signature (`E0803`), so adding the parameter
later would touch every `Debug` impl and every derive expansion in
existence; carrying it now costs one unused binding in hand-written bodies
and one ignored parameter in the expansion. It also positions the alternate
form — Rust's `{:#?}` pretty-printing is exactly an `alternate` flag read by
a `Debug` impl — without a future change having to break the trait to get
it. `types/traits.md` states that today's structural form does not read the
parameter, so no behavior is implied by carrying it.

### `fn debug(self): string` — rejected

Marginally tidier today; breaking tomorrow. The change's own migration
story (rename, add a parameter) would be run again by every program that
adopted the slim signature.

## `toString` is derived, never written

**Decision.** Every `Display` impl makes `value.toString()` available,
defined as `display` at the default `Format` (`precision: null`,
`alternate: false`). The trait does not declare `toString`, so an impl that
writes one is `E0804` — the same code that guards every other trait's
member set — and dispatch of `toString` travels through the impl exactly as
`display` does.

`E0804` here is not a borrowed meaning waiting for a better code: it is its
trigger stated literally — an impl declares a method the trait does not —
and `Display` genuinely declares only `display`. For the case to stop
arising, `Display` would have to declare `toString`, which the language
cannot express today: a trait method has no default body, so declaring it
would oblige every impl to re-provide it (`E0802`), the precise opposite of
derivation. Rule and code are stable as paired; no migration note attaches
to the code.

`ToString` remains a **reserved name** (`E0106`): the method vocabulary
`toString` belongs to the language, and a user-declared `trait ToString`
would hijack the word while resolving to nothing in every rule that used to
name it. As a *reference*, `ToString` now resolves nowhere: `impl ToString
for T` is `E0101`, an unresolved trait name. No new code is added for
migration; the proposal documents the rewrite.

### Drop the name entirely (un-reserve `ToString`) — rejected

Would let a program declare `trait ToString` beside a prelude-provided
`toString` method — two vocabulary systems for one word — for zero gain.

### Keep `ToString` implantable as an alias of `Display` — rejected

Two names for one impl surface is the confusion this change removes; an
"impl either one" rule also needs a coherence story (what happens when both
are written) that the language does not otherwise need.

## Derive re-target

### `#Derive(Debug)` registered; `Display` never derived — RECOMMENDED

**Decision.** The registry's one registered target becomes `Debug`;
`Display` and `ToString` are reserved, unregistered names (`E0111`).

The structural rendering `#Derive(ToString)` automates — `User(name: Alice,
age: 30)` — was always Debug's output wearing the wrong trait's name: field
names, declaration order, no author's intent. `Debug` is the trait whose
*definition* is that text, so the expansion lands on the right contract, and
the reservation of `Display` records the complementary rule: **user-visible
text is written, not defaulted.** Rust has no derive for `Display` either;
the common case (a quick structural dump) is served, and the custom case
(`Welcome, ${user.name}`) is the hand-written impl it always was.

Output continuity is the migration promise: the generated text for
base-type and table-covered fields is byte-identical to today's derive.

### `#Derive(Display` `)` structural — rejected

This is the status quo's mistake named honestly: generating internal
structure as the default *user-facing* string form is what this change
undoes. A derived `Display` would put `User(name: Alice, age: 30)` back
into `${user}`.

### Both registered — rejected

Offers the author the choice between two expansions that produce the same
text under two trait names — the exact ambiguity a closed registry exists to
avoid.

### Field rules: the output table, then `Debug` — RECOMMENDED

**Decision.** The expansion renders each field as:

1. a value of any type other than a `struct` or an `enum` renders directly,
   with **no implementation required**, exactly as `print` renders it — a
   base type as itself, a container from its elements, a fixed-bit numeral
   as its numeral;
2. a value of an `enum` type renders through its `debug` when it implements
   one, and otherwise by the output table's enum row — the same fallback
   `print` makes (the table has always shown enums with no impl);
3. a value of a `struct` type renders through its `debug`; a struct with no
   `Debug` impl fails the derive with `E0801`,
   `` `User` does not implement `Debug` `` — the message shape `E0801`
   already owns, and a struct is the one kind with no rendering of its own;
4. a value typed `unknown` — in a field or in any structural position —
   fails the derive with `E0212` rather than rendering: no
   expansion can be chosen for a type the language does not know
   ([decision](#unknown-is-rejected-not-rendered)).

The rules are recursive through **structural positions** — a container's
elements, an optional's `T`, a tuple's elements, a `Result`'s `T` and `E`
slots, an enum's payloads, the term defined with the output table — so
`List<User>`, `User?`, `(User, int)`, and `Result<User, string>` each
reach `E0801` at `User` when `User` has no `Debug`; nothing a type holds
launders what it holds (`List<SomeEnum>` without a `Debug` falls back to
the enum row instead, per the second rule). A wrapper type
is never covered *as a wrapper*: `struct W { xs: List<int> }` derives
because its field renders directly, while a `W` nested as another derive's
field needs `W`'s own `Debug`. (A *generic* wrapper —
`struct W<T> { xs: T }` — cannot derive at all: that is `E0112`, whose
rationale is exactly that no bound expresses "is not a `struct`, or
implements `Debug`".)

**Expansion mechanics — type-aware, as it always was.** No new compiler
capability class enters with the recursion: the *existing* derive is
already type-aware, or it could never emit its `E0801` (a field-type trait
query) or its `E0805` (an impl collision); attributes are checker-interpreted
precisely because there are no macros, and recursive traversal of field
types at declaration checking already exists for the cyclic-field rule
(`E0113`). This change widens the predicate and lets the traversal pass
through structural positions; it does not open a new mechanism.

Because the body must reach types that have no method to call, the
generated code is specified by the rendering rules above, **not as template
source an author could have written**: a base field's text comes from the
output table (an `int` has no `debug` to call), and a container field has no
template spelling at all (`${xs}` is `E0212`). Direct rendering and a
`debug` call are therefore different code, and only the `debug` call —
used for a `struct`, and for an `enum` that implements one — receives the
forwarded `format`:

```xulo
// #Derive(Debug) on `struct Order { id: int, user: User }`, in effect:
impl Debug for Order {
  fn debug(self, format: Format): string {
    `Order(id: ${self.id}, user: ${self.user.debug(format)})`
    //          └ table-covered: table rendering      └ user-declared: `debug`, format forwarded
  }
}
```

A hand-written `Debug` may ignore `format` without any diagnostic: a
parameter is never reported as unused (the note under the warnings table of
[`type-system/errors.md`](../../type-system/errors.md)), so there is no
`_format` discipline to learn — an author who wants the conventional name
may still write `_format` or `f`.

The old rule was base-types-or-`ToString`, which rejected a `List<int>`
field (`E0801`, `` `List<int>` does not implement `ToString` ``). For a
*structural* rendering that rejection never made sense: `[1, 2]` is exactly
what the table prints for a list, with nobody's cooperation, and Rust's
analogue (`#[derive(Debug)]` on a struct containing a `Vec`) works for the
same reason — there it because `Vec: Debug`, here because the table covers
containers with no impl at all. `E0801`'s domain **shrinks** to
"a `struct` without `Debug`", which is the only kind the expansion
cannot reach.

### Keep the strict base-type-only rule — rejected

Preserves an error that blocked the derive's most useful structural output
for no structural reason, and would make `#Derive(Debug)` weaker than
`print` of the same value.

### Strings render raw — RECOMMENDED (vs. Rust-style quoting)

**Decision.** A `string` field renders its characters unquoted, exactly as
today's derive and the output table do — `User(name: Alice, age: 30)`.

Quoting (`"Alice"`) is Rust's `Debug` choice and is defensible for a
debugging view; rejected here because (a) it breaks the migration promise —
this change's derive must render the same bytes for existing programs — and
(b) it would desynchronize derive from the output table, which prints
strings unquoted in every row. If a quoted form is wanted for the debug
channel later, it is a rendering-table question for the change that owns
`print`'s debug path, not a derive question.

## `print` consults implementations

### `Display`, then `Debug`, then the table — RECOMMENDED

**Decision.** In the output chapter, a value of a user-declared type — a
`struct` or an `enum` — renders through its `Display` at the default `Format`
when it implements one, otherwise through its `Debug`; an `enum` with
neither still renders by the table's enum row (which never needed an
impl), and a `struct` with neither — previously **unspecified**, the table
having no struct row — is `E0212`.

- The precedence is *user-facing first*: Go's `fmt` honors a `Stringer`
  under `%v`, Python's `print` goes `str` then `repr`; a program that wrote
  a `Display` means it, including at `print`.
- It closes a real hole with a real error: `print` of a bare struct was
  neither specified nor diagnosed, and a compile-time complaint (Rust's
  `println!("{}", x)` without `Display` behaves the same) beats an
  accident. The message hint says `implement Display` — true for `${…}` as
  well, which the type still needs; a `Debug`-only type prints fine, and
  `E0212`'s trigger text is widened to name `print`, with two conditional
  notes carrying the rest (position at `print`; the type's own `Debug`).
- It makes `Debug` observable inside the language: a derived struct prints
  structurally (`print(s)` → `User(name: Alice, age: 30)`), which is what
  the re-targeted derive needs and what the standard library's engine will
  later reach as its own verb.
- The delta states the outcomes as a three-row table — the value's type ×
  implements `Display` / implements `Debug` / neither — so the struct/enum
  asymmetry is read off the page, not assembled from prose.

### Completing the output table — RECOMMENDED (required by the field rules)

**Decision.** The rendering table gains rows for every kind the table
missed: `number` (as its runtime member), the fixed-bit numerics
(`i8` … `u64`, `usize`, `isize`, `f32`, `f64`), `unit`, `T?` optionals,
`Range<T>`, `Result<T, E>`, tuples, `Error`, `Format`, and `Ptr`. Each
rendering mirrors a convention the table already sets —
`unit` parallels `null` spelling its own name,
`Ok(v)` follows the enum row's `E::V(…)`, a tuple follows the list's
parentheses-and-commas, a fixed numeral follows the `int` row, `f32`
follows the `float` row's shortest-reads-back rule, `Ptr` joins
`<function>`'s angle brackets (and stays deterministic — no addresses).

**`unknown` gets no row — it is rejected instead** ([decision below](#unknown-is-rejected-not-rendered)).

Two forces make this part of this change rather than a nicety:

1. **The derive's field rule must be total.** Its third case — "any other
   type renders directly" — is a lie for `struct S { x: i32 }` until the
   table names `i32`; and the old rule was no better (`i32` failed
   `#Derive(ToString)` too, because `i32` has no string form anywhere in
   the language).
2. **`print` already promises what the table does not deliver.** Its
   signature accepts any type and its prose says it renders structured
   values directly, yet `print(Time::now())`, `print(Ok(1))`,
   `print((1, 2))`, and `print(unitValue)` matched no row and no rule —
   pre-existing holes the completed table closes on the way past.

### Failing the derive on rendering-less types — rejected

The alternative was to leave the gaps and let the field rule fail them
(`E0801`, `` `i32` does not implement `Debug` ``). Rejected because the
message has **no actionable path**: an `impl Debug for i32` body cannot
produce text at all — both `${…}` and `str` reject `i32` — so the user
would be told to do the impossible. Fixing the table is the only repair
that keeps the diagnostic honest, and it is the repair `print` needed in
its own right.

### `unknown` is rejected, not rendered — RECOMMENDED

**Decision.** No rendering crosses an `unknown`. Three sites enforce one
rule, all with `E0212` and the message interpolation already gives:
`print` on an `unknown` — as the operand or in any structural position;
`#Derive` on a field or position typed `unknown`; and
`${…}`/`str`, which have always rejected it. `unknown` is a *known* static
type, so the check needs no timing rule of its own: it decides at the
call, or at the expansion — including the instantiation call that supplies
`T := unknown` to an otherwise unbounded generic body.

Rendering it at run time instead was the alternative, and three walls stop
it:

1. **Impl dispatch is static.** No runtime type information is consulted
   ([`../../types/traits.md`](../../types/traits.md) — the target is
   fixed before the call), so a struct value behind `unknown` cannot reach
   its `Display` or `Debug` — neither the "print prefers" path nor the
   fallback exists.
2. **Struct shape is compile-time metadata.** Field names and the
   `S(f: …)` form live with the declaring type; a runtime tag carries the
   kind, not the vocabulary a rendering would print.
3. **The value has no string form anyway.** Interpolating or `str`-ing the
   same value is `E0212` today — granting `print` a dynamic rendering the
   string-form positions refuse would make the channels disagree about
   what the language knows.

The reviewer-facing rule of thumb: **narrow before you show.** `unknown`
is the type that admits nothing without narrowing; a rendering request is
a use like any other.

### `Debug` always for user types — rejected

Makes `print` a pure diagnostic view, but then a type with only `Display`
(the common hand-written one) becomes *unprintable* while `${…}` renders it
fine — an inversion every Go/Python reader would trip over — and it breaks
today's `print`-honors-`Stringer`-style expectation the other direction.

### Leave `print` alone — rejected

Would land `Debug` with no in-language consumer at all: the re-targeted
derive's output could not be printed, `str`, and `${…}` refuse it, and the
spec would ship a trait whose text exists only in another document's verbs.
The struct row's silence is a hole this change already has to walk around;
filling it costs one paragraph and one `E0212` trigger.

### Preference of `Display` over `Debug` when both exist — noted consequence

A type with both prints its `Display`; its `Debug` is unreachable *from
`print`* until an engine verb reaches it (the standard library's formatter).
Accept: `print` favors the user-facing text, and the structural view is
recovered through `str`-independent channels when they exist — Go behaves
the same under `%v`.

The mirror case is equally intended: a `Debug`-only type prints (`print(u)`
shows the structure) but does **not** interpolate (`` `${u}` `` is `E0212`).
That is Rust's shape exactly — `{:?}`-capable, `{}`-incapable — and the two
channels are supposed to be answerable separately; an author who wants both
everywhere derives `Debug` beside a hand-written `Display`. Recorded for
migrations in the [proposal](proposal.md#compatibility).

The same asymmetry governs structural positions: `print` of a `List<User>`
renders each element by this preference order (`Display` first), while a
derived `Debug` whose field is a `List<User>` reaches `User`'s `debug`
only — print asks each value for its best rendering, derive asks every
value for structure and never consults `Display`. The two channels are
internally consistent; they are not identical, by design.

## Grammar and typing consequences

- **Grammar: none.** `Display`, `Debug`, and `Format` are identifiers in
  positions that already exist (trait names, type names); no production, no
  keyword, no lexical change, and the production count is untouched.
- **Typing.** `Format` is an ordinary nominal struct of the prelude —
  assignable only to itself, constructible by literal, fields `int?` and
  `bool` under the ordinary rules. `precision ?? 1` uses the existing
  null-coalescing rule; no coercion is added (a `Format` converts nowhere).
- **Bounds.** `T: Display` and `T: Debug` are ordinary trait bounds;
  `E0112`'s restriction survives because no bound means "is not a `struct`
  *or* implements `Debug`" — the disjunction the
  expansion needs cannot be written.
- **Dispatch.** `display`, `debug`, and the derived `toString` obey
  module-scoped impl selection unchanged; ambiguity is `E0809` as for any
  member call. The derived `toString` joins member resolution through the
  rule the delta adds to `Member calls` (found when neither existing step
  finds a candidate; yields to inherent and declared members), so an
  unbounded `T` still has no `toString` — only a `T: Display` bound grants
  it, exactly as bounds grant declared members today.
- **`print`'s check timing.** The struct-with-neither `E0212` is raised
  where the operand's type is concrete — at the call supplying the type
  argument for a generic operand, the site that already checks bounds
  (`E0801`). This is deliberately *not* interpolation's regime: `${x}` on
  an unbounded `T` fails at the definition because `T: Display` is a bound
  the language can demand, while `print`'s acceptance is wider than any
  bound can state (base types and containers satisfy no bound the language
  has), so its check follows the operand to where the type becomes known.

## Diagnostics consequences

**No new codes.** Text changes only:

| Code | Change |
|------|--------|
| `E0109` | example becomes `#Derive(Debug) fn main()` |
| `E0111` | trigger and example unchanged (`#Derive(Display)` stays a reserved target); the note becomes `` registered targets: `Debug` `` |
| `E0112` | example becomes `#Derive(Debug) struct Box<const N: int> { v: int }`, message `` cannot derive `Debug` for a generic type `` |
| `E0212` | trigger adds `print` and a `#Derive` expansion; example `` `${user}` `` without `Display`; message `` `User` has no string form; implement `Display` ``; two conditional notes (at `print`; when the type implements `Debug`); `unknown` is one trigger at every site, always with the same message |
| `E0801` | row unchanged (trait-agnostic); its derive use narrows to "user-declared type with no `Debug`" via the traits chapter and the errors footnote — and the footnote's stray "template positions" clause is corrected: the expansion's field rule owns `E0801`, template interpolation owns `E0212` |
| `E0805` | row unchanged (trait-agnostic); the footnote names `impl Debug` |
| `E0804` | unchanged, newly *exercised*: an `impl Display` declaring `fn toString` |
| `E0101` | unchanged, newly *exercised*: `impl ToString for …` no longer resolves |
| `E0106` | unchanged; its prelude list gains `Format`, `Display`, `Debug`, keeps `ToString` |

## Migration

```xulo
// before                                // after
impl ToString for User {                 impl Display for User {
  fn toString(self): string { … }          fn display(self, format: Format): string { … }
}                                        }

#Derive(ToString)                        #Derive(Debug)
struct User { name: string, age: int }   struct User { name: string, age: int }

u.toString()                             u.toString()          // unchanged
`${u}`                                   `${u}`                // unchanged (Display now)
str(u)                                   str(u)                // unchanged
print(u)                                 print(u)              // now specified: Display first
```

Two deliberate movement rules: a field type that had only `ToString` must
gain `Debug` for the derive to accept it, and containers in derive positions
become accepted (they were rejected; the table renders them). One rough
edge is accepted: `impl ToString for …` reports `E0101` (unresolved name)
rather than a tailored message — a dedicated migration code was weighed and
rejected as a code spent on a rename the proposal documents, and a tailored
hint inside `E0101`'s row was rejected alongside it: a chapter row must
stay timeless once the rename's migration wave passes, while migrations are
read in the proposal — and, as narrative, in the website layer
([`README.md`](../../README.md), § 6).

## Impact

- **Eleven chapters** fold from [`specs/`](specs/): the prelude pair,
  `traits.md` (the largest delta: contracts section, registry, derive),
  `intrinsic-functions.md` (`str` signature and prose, the `print`
  paragraphs and precedence table, the output table's completion rows, and
  an example), the `str`-domain statements in five further chapters, the
  errors table and
  footnotes, and one naming example.
- **Follow-ups owned elsewhere:** the standard library specification's
  formatting module (verb dispatch to `Display`/`Debug`, `Format`
  construction from directives — cited after acceptance); template format
  specifiers (a future grammar change desugaring onto `Format`).
- **Non-impact:** grammar files, AST, lexical tables, `ast.md`, and the
  machine-readable count are untouched.

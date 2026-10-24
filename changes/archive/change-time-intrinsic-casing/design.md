# Design: official names are normative; `Time` casing

## The decision

`names.md` gains one paragraph after the RECOMMENDED preamble: the
conventions table is **normative for every name this specification
defines itself** — built-in types, protocols, namespaces, prelude
operations, and intrinsic functions — while user code keeps the
recommendation with no diagnostic. The standard library is noted as
held to the same table (informative; its chapters live outside this
specification).

Two renames follow immediately: `Time::now_nanos` →
`Time::nowNanos`, `Time::monotonic_nanos` → `Time::monotonicNanos`,
in the Time table, both prose bullets, the elapsed-time example, and
the complete signature index.

## Alternatives considered

### Do nothing — casing is just style anyway

**Cost.** The names are the API: two conforming prelude
implementations would expose different spellings of the same member,
and the divergence is observable at every call site. "Style only"
holds for user code precisely because user code has one author; the
built-in surface has two readers and zero freedom. Rejected — this is
the defect being fixed.

### Rename, but keep the recommendation purely advisory (no names.md sentence)

**Cost.** Fixes today's two names and re-opens tomorrow's: nothing
states that official names obey the table, so the next new intrinsic
can drift again and the next reviewer must re-derive the rule from
precedent. The sentence costs two lines and makes the sweep a rule
instead of a habit. Rejected.

### Enforce the table on user code too (a naming diagnostic)

**Cost.** Contradicts the explicit ruling that naming is convention
and not enforcement, breaks `names.md`'s "produces no diagnostic"
guarantee, and hands the compiler a style police it must also
implement. Rejected outright.

### Scope the normative sentence to functions only

**Cost.** The table has six rows and built-ins occupy several of them
(`List`, `Result`, `ToString`, `Math`). A function-only sentence would
leave the types half-covered and need a second sentence later. The
whole table, one sentence. Rejected.

### The chosen shape — normative table for official names, two renames

**Why.** It draws exactly one line — official vs user — where the user
already drew it: convention for programs, certainty for the language's
own surface. The checkable deliverable is precise: zero snake_case
names left among built-ins (a greppable property), and no new
diagnostics anywhere.

## Consequences

### Names

- `names.md`: one new paragraph; the RECOMMENDED sentence and the table
  itself are unchanged.
- Built-in inventory after the change: every intrinsic and prelude
  operation is lowerCamelCase, every built-in type/protocol
  UpperCamelCase — including `Time::nowNanos` and
  `Time::monotonicNanos`.

### Typing and runtime

None: signatures, semantics, and evaluation are untouched; only the
spellings change.

### Diagnostics

| Code | Change |
|------|--------|
| (new codes) | none — the rule is normative about spellings, not a check |

### Grammar

None: identifiers are unchanged as a token class; two member spellings
in examples and tables move to their conventional form.

### Parity

The change adds one paragraph of reading; in exchange it removes a
snake_case spelling from a closed official inventory and pins that
inventory's casing rule — a greppable, checkable property. No syntax
appears, so the gate's syntax clause is untouched.

# Design: the struct composition cycle rule

## The decision

A struct's field types compose into the struct's value. The declaring type
MUST NOT be reachable from its own fields through field positions, tuple
elements, and alias expansion without an intervening indirection — an
indirection being `List<T>`, `Map<K, V>`, `Set<T>`, `T?`, a function type,
or an enum type. A violation is a compile-time error (`E0113`). Enum
payloads may mention the enum being declared directly; payload recursion
is well-formed and needs no indirection. The tuple rule falls out of the
field rule: a tuple element is a mandatory position exactly like a field.

## Alternatives considered

### No rule — the status quo

**Cost.** This is the defect being fixed, not an alternative:
`struct Node { node: Node }` is accepted today with no consequence stated
anywhere, and an implementer must invent one. The declaration families
around it (`E0105` aliases, `E0106` reserved names, `E0103` duplicates)
are all closed; struct composition is the hole. Rejected.

### Full inhabitance analysis, enums included

Walk the whole type graph — structs, enums, aliases, generics — and
reject exactly the types with no constructible values, mutual struct–enum
loops included (`struct A { e: E }` with `enum E { V(A) }`). **Cost.**
The analysis needs a fixpoint over possibly mutually recursive graphs,
with generic instantiation folded in, before a single struct declaration
can be accepted — an unbounded computation attached to a well-formedness
check whose siblings are single-lookup decisions. It also rejects
`enum E { V(E) }`, which the chosen rule leaves permitted as an explicit
non-goal (it is uninhabited but harmless: nothing can construct it, and
nothing must). Rejected for v1; the syntactic cycle rule catches the
cases programs actually write.

### Reuse `E0105` — extend the alias rule

**Cost.** `E0105`'s trigger is "a type alias that expands to itself" and
its message names the alias. Struct composition is a different
declaration form with a different message; stretching one code across
both weakens the closed code table's one-code-one-trigger discipline.
Rejected: `E0113` is free, and the declaration family is numbered for
exactly this growth.

### Mandatory-position cycle rule, enum as break — chosen

**Why.** It has the shape the alias rule already teaches: mandatory
positions compose valuehood, constructors provide indirection, a cycle
through mandatory positions alone is ill-formed. The check is bounded —
one struct's expansion graph, no enum fixpoint — and every accepted
program's meaning is unchanged. Enum payloads break the walk because
enums are the language's own recursion device; forbidding
`enum Tree { Leaf(int), Node(Tree, Tree) }` would delete the canonical
tree from the language. The checkable deliverable is precise: one new
compile-time condition at Layer 1, closed under the error table, with the
enum permission stated so neither side contradicts the other.

## Consequences

### Well-formedness

- `struct Node { node: Node }`, `struct A { b: B }` / `struct B { a: A }`,
  `struct S { t: (S, int) }`, and `struct S { xs: SL }` with
  `type SL = S` are `E0113`.
- `struct Node { next: Node?, kids: List<Node> }`,
  `struct S { f: fn(S): unit }`, and `struct A { e: E }` with an enum `E`
  whose payloads mention `A` are well-formed.
- Type parameters are opaque to the walk: `struct Pair<A, B> { first: A,
  second: B }` reaches nothing.

### Typing and constructors

- No change to construction, field access, mutability, visibility, or
  nominal identity. `E0205` (missing field) and `E0204`/assignment rules
  are untouched.
- `E0105` is untouched: alias cycles keep their own code and message.

### Diagnostics

| Code | Change |
|------|--------|
| `E0113` | new: a struct whose field types reach the declaring struct through fields, tuple elements, and alias expansion with no intervening indirection — message ``type `Node` has a cyclic field definition`` |
| `E0105` | unchanged — aliases only |

### Grammar

None: no production, token, or declaration-form changes.

### Parity

The change adds one rule sentence and one error row of reading; in
exchange it adds a new statically enforced property (uninhabited-by-
construction struct declarations are rejected) — the gate's Layer 1 form.
It also removes an unspecified corner from a declaration family the
specification otherwise closes completely, which the gate explicitly
welcomes.

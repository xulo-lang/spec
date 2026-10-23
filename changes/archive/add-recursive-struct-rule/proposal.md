# Change Recursive Struct Rule

**Status:** accepted · **Date:** 2026-10-23 · **Affects:**
`types/composite-types.md`, `types/enums.md`,
`type-system/errors.md`, `type-system/checking-rules.md`,
`type-system/README.md`

## Summary

Nothing in the specification stops a struct from declaring a field of its
own type — `struct Node { node: Node }` — or from closing a loop through
other structs and aliases — `struct A { b: B }` with `struct B { a: A }`.
Such a type admits no constructible value: every construction would demand
one construction of the same type first. The language has exactly one rule
for this class of defect already — an alias that expands to itself is
`E0105` — but structs walk through nothing. This change extends the
family:

> A struct's field types MUST NOT reach the declaring struct through field
> positions, tuple elements, and alias expansion without an intervening
> indirection (`E0113`); recursion through `List<T>`, `Map<K, V>`,
> `Set<T>`, `T?`, a function type, or an enum payload is well-formed.

`E0113` is the next free code in the declaration family
(`E0101`–`E0112` are taken). Enum payload recursion — `enum Tree {
Leaf(int), Node(Tree, Tree) }` — is explicitly permitted, with no
indirection required. The deltas in [`specs/`](specs/) are the text the
chapters gain once the change is accepted.

## Motivation

### A silent hole on a closed-checking family

`type-system/checking-rules.md` promises that names resolve, signatures
annotate, and aliases terminate — and struct composition is a declared
type well-formedness question the checker never asks. An implementer
facing `struct Node { node: Node }` today must invent a verdict: accept
it silently (and let every constructor be unwritable), or reject it with a
code they make up. Neither matches the specification's claim that
well-formedness is defined, not left to implementations.

### Self-hosting needs the same rejection everywhere

Bootstrapping the compiler written in Xulo against the Rust
implementation requires both to agree on which struct declarations are
ill-formed. The cycle `struct Node { node: Node }` is the natural
transcription of a linked-list node by anyone porting data-structure code;
without a rule, one compiler accepts and the other may not — a
conformance divergence on a construct as basic as a struct declaration.

### The alias rule is a precedent that stops one step short

`E0105` already rejects `type A = B` with `type B = A`, and explicitly
permits recursion through a constructor — `List<T>`, `T?`. Struct fields
need the same treatment: a mandatory position with no constructor in
between is a cycle; an indirection makes it inhabited. The rule reuses
that shape rather than inventing a new one, and reuses the next free
declaration-family code rather than opening a new range.

## Scope

**In scope.** The struct composition rule and its diagnostic `E0113`; the
enums-side permission for payload recursion; the checking-rules restatement;
the `type-system/README.md` summary row.

**Out of scope.** Any inhabitance analysis of enums (a struct–enum mutual
loop is not analyzed); changes to `E0105` or alias rules; a cycle rule for
generic instantiation; recursion through tuple types alone is covered by
the field-position rule, not by a separate tuple rule.

## Compatibility

No conforming program can regress: the rule rejects only programs that
admit no constructible value and were previously unspecified — an
implementation that accepted `struct Node { node: Node }` accepted a type
no program could construct. Ordinary recursive types (`struct Node {
next: Node?, children: List<Node> }`) are untouched, as they already must
be for any list or tree in real code.

## Non-goals

- No fixpoint inhabitance computation — enum payload recursion is
  permitted as written, single-variant loops included.
- No change to how `Result<T, E>` and other enum-based indirection behave;
  they are already the sanctioned way to terminate a recursive shape.

## Readiness

- [x] Depends on nothing unaccepted; every delta is written against the
      current, folded chapters.
- [x] All five affected passages are covered by [`specs/`](specs/) and
      [`tasks.md`](tasks.md), including both the error table and the
      checking-rules enumeration.

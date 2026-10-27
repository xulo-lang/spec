# Change Proposals

This directory tracks proposed changes to the Xulo Language Specification in
the OpenSpec style. A proposal describes the specification as it *would* read
after the change; it does not amend the chapters themselves. Until a change is
accepted and folded in, the main chapters remain normative: a construct that
appears only in a proposal is not part of the language, and where a proposal
and a chapter disagree, the chapter governs.

## Workflow

Every change passes through the same stages:

- **Draft.** The change directory exists with the three required artifacts —
  `proposal.md`, `design.md`, and at least one file under `specs/` — and the
  [index](#index) records it with status `draft`. A draft MAY be edited freely.
- **Review.** The proposal is being evaluated: the alternatives in `design.md`
  are weighed against the chosen design, the deltas in `specs/` are checked
  against the chapters they claim to replace, and the scope is confirmed. The
  index records the status `review`.
- **Accepted.** The deltas are folded into the chapters they name, so that the
  replacement text becomes chapter text; the index entry is dropped and the
  directory is moved to `archive/` with status `accepted`.
- **Rejected.** The directory is moved to `archive/` with status `rejected`,
  recorded by a one-line reason in its `proposal.md`, and is never folded in.

Two rules hold at every stage:

1. **Unaccepted changes MUST NOT contradict the main chapters.** The current
   chapters are normative until acceptance; a proposal states its deltas as
   future text, never as present fact.
2. **A change is folded in whole or not at all.** Partial folding would leave
   the chapters inconsistent with one another, so a change that cannot be
   applied completely stays in draft.

## Directory layout

Each change lives in one directory named after its change ID:

```text
changes/
├── README.md                 workflow, artifact rules, change IDs, index
└── <change-id>/
    ├── proposal.md           required: what the change does and why
    ├── design.md             required: alternatives, choice, rationale
    ├── specs/                required: normative deltas
    │   └── <chapter>.md      one file per affected chapter
    └── tasks.md              optional: the edits an acceptance requires
```

- Change IDs are **kebab-case** (`add-explicit-type-arguments`), name the
  change rather than its motivation, and are never reused.
- `proposal.md`, `design.md`, and `specs/` are required, and `specs/` MUST
  hold at least one file. `tasks.md` is optional.
- A file under `specs/` is named after the chapter it modifies — `generics.md`
  for [`../types/generics.md`](../types/generics.md) — so that a reader can
  match every delta to its chapter at a glance.

## Artifact rules

| Artifact | Contents |
|----------|----------|
| `proposal.md` | **What** the change adds, removes, or alters; **why** it is wanted, argued from examples; the **scope** — what is in and what is out. Carries the status metadata and no normative text. |
| `design.md` | The alternatives considered with their costs, the design actually chosen, and the rationale for choosing it, including the grammar, typing, and diagnostic consequences. |
| `specs/` | The exact normative deltas: post-acceptance replacement text, one file per affected chapter. |
| `tasks.md` | Optional checklist of the cross-references and neighbouring passages an acceptance must update. |

Each file under `specs/` follows one convention. Its first heading is the
target chapter's own heading followed by a delta marker, and the line after it
is a blockquote naming the file the delta replaces:

```markdown
# Generics — Delta: Explicit Type Arguments

> **Delta** — replaces the “Explicit type arguments” paragraph in `types/generics.md`.
```

The `> **Delta** — …` blockquote is required and MUST name the target file.
The body then gives the post-acceptance text in full, so that folding the
change in is a matter of copying the body into the chapter it names.

## Change ID conventions

A change ID begins with a verb phrase that states the direction of the edit:

| Prefix | Use |
|--------|-----|
| `add-` | a construct, rule, or chapter that does not exist yet |
| `remove-` | a construct, rule, or chapter that is deleted |
| `change-` | a construct, rule, or chapter that is altered in place |

The remainder of the ID is kebab-case and names the language element touched,
never a version number or an implementation detail.

## Index

| Change | Status | Summary |
|--------|--------|---------|
| [change-collection-members/](change-collection-members/proposal.md) | draft | List is the growable sequence with Array-style dynamic and callback members; Map/Set have no key subscript or member form (`.get`/`.set` only); sequences use `.len`, containers `.size`; the prelude collection/string functions are removed. |
| [add-const-generics/](add-const-generics/proposal.md) | draft | const parameters `<const N: int>`; mixed type/const generic lists; explicit const arguments (integer literals, module consts, const parameter names); no const arithmetic or inference; `E0812`/`E0813`; a fixed-array consumer is a separate change. |
| [add-fixed-array/](add-fixed-array/proposal.md) | draft | builds on `add-const-generics` and `change-collection-members`; inline `Array<T, N>` with a const N, context-typed literal, `Array::filled`, read/callback/sort members, value copy, no dynamic members; depends on both prerequisites. |

## Archived

Accepted and rejected changes leave the index above and are kept in `archive/`
with their final status.

| Change | Status | Summary |
|--------|--------|---------|
| [archive/add-declaration-attributes/](archive/add-declaration-attributes/proposal.md) | accepted | `#Derive(ToString)` — declarations MAY carry `#Name` attributes from a closed, checker-interpreted registry; `E0108`–`E0112` registered. |
| [archive/add-foreign-function-interface/](archive/add-foreign-function-interface/proposal.md) | accepted | `extern "C" { … }` imports and `pub extern "C" fn …` exports behind a closed ABI registry (`"C"`, `"system"`); ABI-safe signature whitelist, opaque non-null `Ptr`/`Ptr?`, `usize`/`isize`; `E0901`–`E0903` registered; zero-UB claim gains a foreign-boundary carve-out. |
| [archive/add-explicit-type-arguments/](archive/add-explicit-type-arguments/proposal.md) | accepted | a call MAY write explicit type arguments — `first<Int>(…)` — which replace inference; `E0807`/`E0808` registered. |
| [archive/add-test-title/](archive/add-test-title/proposal.md) | accepted | `#Test("parse test") fn …` — a test MAY carry a parenthesized string title; the identifier stays the identity, bare `#Test` stays canonical. |
| [archive/add-reserved-word-production/](archive/add-reserved-word-production/proposal.md) | accepted | define the `ReservedWord` production that `Token` already references, as a lexical condition over the reserved list of `lexical-structure.md`, and index it — restoring the "defined exactly once" invariant (191 → 192 productions). |
| [archive/add-recursive-struct-rule/](archive/add-recursive-struct-rule/proposal.md) | accepted | a struct's field types MUST NOT reach the declaring struct through fields, tuple elements, and aliases with no indirection (`E0113`); enum payload recursion is explicitly permitted. |
| [archive/add-exit-status/](archive/add-exit-status/proposal.md) | accepted | pin the exit status of ordinary termination — 0 on a normal exit, nonzero on a runtime failure; exact nonzero values stay implementation-defined; no `exit` intrinsic. |
| [archive/add-string-introspection/](archive/add-string-introspection/proposal.md) | accepted | four prelude functions — `stringSize`, `stringCodeAt`, `stringSlice`, `stringBytes` — fulfill the dangling "string intrinsics" promise; scalar indices; *String index out of range* joins both closed failure tables. |
| [archive/add-wrapping-arithmetic/](archive/add-wrapping-arithmetic/proposal.md) | accepted | eight `Math` intrinsics — `wrappingAdd`/`Sub`/`Mul` (mod 2⁶⁴, never trap), `popCount`, `countLeadingZeros`/`countTrailingZeros`, `rotateLeft`/`rotateRight` — close the bit-hashing gap the trap policy leaves. |
| [archive/change-time-intrinsic-casing/](archive/change-time-intrinsic-casing/proposal.md) | accepted | official names are normative under the `names.md` table; `Time::now_nanos`/`monotonic_nanos` → `nowNanos`/`monotonicNanos`; user code stays RECOMMENDED with no diagnostic. |
| [archive/align-intrinsic-casing/](archive/align-intrinsic-casing/proposal.md) | accepted | Rename the multi-word prelude intrinsics and `to_string` to lowerCamelCase (`mapGet`, `setInsert`, `listRemove`, `toString`), matching names.md. |
| [archive/change-module-system-to-trees/](archive/change-module-system-to-trees/proposal.md) | accepted | a package is a `mod`-declared tree; `::` paths, manifest package names (one per dependency), verbs `use`/`pub use` replace string specifiers. |
| [archive/change-shift-count/](archive/change-shift-count/proposal.md) | accepted | pin the shift-amount domain (`0 … width−1`): a constant violation is `E0210`, a non-constant one a runtime trap — the same shape as division by zero; shifts in range never overflow. |
| [archive/change-test-to-attribute/](archive/change-test-to-attribute/proposal.md) | accepted | the test declaration is `#Test [async] fn name() { … }` from the attribute registry; `test` leaves the keyword set (42 → 41). |
| [archive/lowercase-primitive-keywords/](archive/lowercase-primitive-keywords/proposal.md) | accepted | Primitive value types become lowercase keywords (`bool`, `string`, `int`, `i8`…`f64`, `unit`, `unknown`); `View`, `Range<T>`, composites, and user types stay UpperCamelCase; 41 → 58 keywords. |

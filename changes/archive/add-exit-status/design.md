# Design: exit status of ordinary termination

## The decision

A program that terminates normally — `main` returns `unit`, its returned
task settles, or the presentation target of a `View` program closes —
exits with status 0. A runtime failure that ends the program — a panic,
any condition of the closed failure tables — exits with a nonzero status.
The exact nonzero values, and how the failure is presented to the
environment, remain implementation-defined and MUST be documented. The
rule lives in `memory-and-runtime.md` as its own subsection; the
`main` section of `modules/source-files.md` and the self-hosting note of
`README.md` § 1.1 cross-reference it.

## Alternatives considered

### Leave it unspecified — the status quo

**Cost.** This is the defect being fixed. `memory-and-runtime.md` claims
to close everything the language fixes, with two enumerated exceptions;
the exit status of a successful program sits in neither. Two conforming
implementations disagree about the single bit a harness reads, which
breaks the whole point of a bootstrap conformance check. Rejected.

### Pin exact failure codes — one status per failure kind

Give each closed-table condition its own nonzero code, so a harness can
tell a panic from an overflow. **Cost.** The mapping would need a second
closed table maintained in lockstep with the first, and it promises
diagnostic precision the language does not otherwise have — a panic's
*kind* is already reduced to "there is no handler" by the model. Callers
that need the kind use `Result`; the process boundary is not a typed
channel. Rejected for v1; implementations MAY still use distinct codes
documented per implementation, since nonzero is the only constraint.

### Add an `exit(code)` intrinsic

Let `main` choose its status: `exit(0)`, `exit(2)`. **Cost.** It is an
I/O-adjacent effect the language does not otherwise have (no `print` in
the model either — `print` is an intrinsic of the standard library's
entry, but `exit` additionally abandons unwinding, tasks, and the `View`
loop mid-flight). The FFI already provides it where a C ABI exists
(`extern "C" { fn exit(code: i32): unit }`), and the standard library can
wrap it later without a language change. Rejected from the language
surface; noted in the self-hosting note as the FFI route.

### Pin 0 and nonzero — chosen

**Why.** It is the minimum every host needs, it matches the one adjacent
fact the spec already has (implementation-defined presentation of a
failure), and it costs one subsection plus two cross-references. Exact
failure codes stay implementation-defined exactly where the
implementation-defined list already puts them — "the exact nonzero exit
status" replaces the old unqualified "exit status" in that bullet. The
checkable deliverable is precise: a new normative Layer 3 rule with a
closed pair of outcomes (0, nonzero), closed under the existing failure
enumerations.

## Consequences

### Runtime model

- `memory-and-runtime.md` gains `## Exit status`, placed after
  `## Runtime failures` (the failures it reacts to) and before
  `## Determinism`.
- The implementation-defined bullet narrows: "message text, stream, and
  the exact nonzero exit status" replaces "message text, stream, exit
  status". The count "four, all at or beyond the boundary" is unchanged.

### Modules

- The `main` section gains one bullet cross-referencing the rule. The
  allowed signatures, `E0608`, and `E0609` are untouched — `main` still
  takes no parameters and returns only `unit` or `View`.

### Diagnostics

| Code | Change |
|------|--------|
| (new codes) | none — exit status is a runtime/host observation, not a diagnostic |

### Grammar

None: no production, token, or declaration-form changes.

### Parity

The change adds one subsection, one bullet, and one informative paragraph
of reading; in exchange it fixes a normative runtime semantics (exit
status 0 vs nonzero) that was previously in neither closed list — the
gate's Layer 3 form. It removes an unspecified behavior from a
specification that claims none, which the gate explicitly welcomes.

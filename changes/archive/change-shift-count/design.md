# Design: the shift-amount domain

## The decision

The shift amount MUST be a non-negative integer strictly less than the bit
width of the operands' type — 64 for `int`, `W` for `usize` and `isize`, the
type's own width otherwise. An amount outside that range is a compile-time
error (`E0210`) when the value shifted and the amount are constants, and a
runtime trap otherwise. Inside the range the bit-fill definition is the
whole value rule: a shift never overflows.

## Alternatives considered

### Zero-fill the value — Go's rule

`x << 64` on a 64-bit type yields `0`, and `x >> 64` yields the sign-fill
(`0` or `-1`). **Cost.** A silently nonsensical value flows on: hash
rounds and bitstream decodes produce wrong output with no stop, which is
precisely the failure mode the language traps for everywhere else —
division by zero does not yield `0`. It also forces the prose to special-case
the amount at or past the width for each direction, growing the rule instead
of letting one domain sentence cover it. Rejected.

### Mask the count — C and Rust-release's rule

`x << 64` behaves as `x << 0`. **Cost.** The worst of the silent options:
the result is a plausible wrong value (`1 << 64 == 1`), and implementations
that follow debug/release conventions split into behaviors the same source
exhibits under different builds — the divergence class the zero-UB claim
exists to eliminate. Rejected.

### Leave it unspecified — the status quo

**Cost.** This is the defect being fixed, not an alternative: a MUST with no
stated consequence, on a construct whose behavior the specification
elsewhere claims to close completely. Two conforming implementations — the
Rust compiler and a self-hosted one — could disagree observably on ordinary
bit-twiddling. Rejected.

### Trap, like division by zero — chosen

**Why.** The spec already has exactly one treatment for operand-domain
errors: constant operands are `E0210`, non-constants are a runtime trap
(`expressions/operators.md`, `type-system/primitive-types` overflow rule,
the runtime-failure tables). Shift-amount-out-of-domain belongs to that
family — an operand outside the operator's defined domain — so it takes the
same rule and reuses `E0210`, gaining no new code. The checkable deliverable
is precise: one new compile-time condition at Layer 1 and one new normative
runtime failure at Layer 3, both closed under the existing enumerations.

## Consequences

### Typing and constant evaluation

- Operand and result types unchanged: one integer type on both sides
  (`E0210` no, `E0201` for mixed types, unchanged).
- Constant folding: `1 << 64` in a constant expression is `E0210` — the
  same evaluation that already catches `1 / 0`. The three passages that
  enumerate constant-arithmetic conditions (`checking-rules.md` twice,
  `coercion.md`, `let-and-assignment.md`) each gain the shift case so the
  enumeration stays exhaustive.

### Runtime

- One new row in each of the two tables that MUST stay identical
  (`memory-and-runtime.md`, `error-handling.md`): *Shift amount out of
  range*, specified in `expressions/operators.md`.
- In-range semantics — fill, discard, `<< 0` identity — untouched. The
  "never overflows" sentence exists only to stop `100i8 << 1` from being
  misread against the overflow rule; it restates what bit-fill already
  determines.

### Diagnostics

| Code | Change |
|------|--------|
| `E0210` | trigger grows: ``constant arithmetic divides by zero, takes a remainder by zero, overflows its type, or shifts by a negative or out-of-range amount`` |
| (new codes) | none — reuse is the point |

### Grammar

None: no production, token, or precedence changes.

### Parity

The change adds one sentence and one table row of reading; in exchange it
adds a new statically enforced property (constant out-of-range shifts are
rejected) and a new normative runtime semantics (non-constant ones trap)
— the gate's Layer 1 and Layer 3 forms together. It also removes an
instance of unspecified behavior from a specification that promises none,
which the gate explicitly welcomes in both directions.

# Change Shift Count Domain

**Status:** accepted · **Date:** 2026-10-23 · **Affects:**
`expressions/operators.md`, `type-system/errors.md`,
`type-system/checking-rules.md`, `type-system/coercion.md`,
`statements/let-and-assignment.md`, `memory-and-runtime.md`,
`error-handling.md`

## Summary

The shift rule says the amount "MUST be a non-negative integer" but never
says what happens when it is not, and never bounds it above: `1 << 64` on
`int` is specified nowhere — it appears in neither the compile-time error
list nor the closed runtime-failure table, both of which
[`../../memory-and-runtime.md`](../../memory-and-runtime.md) presents as
complete. This change gives the shift amount the same shape every other
numeric domain error already has:

> An amount outside `0 … width−1` is a compile-time error (`E0210`) when the
> value shifted and the amount are constants, and a runtime trap otherwise.

It also states what the bit-fill definition already implies — inside the
range a shift never overflows — so `100i8 << 1` cannot be misread as a
trapping overflow. `E0210` gains the shift case; no new diagnostic code
appears. The deltas in [`specs/`](specs/) are the text the chapters gain
once the change is accepted.

## Motivation

### A MUST with no consequence, on a closed list that omits it

The constraint exists (`expressions/operators.md`,
`type-system/checking-rules.md`) but its violation has no defined
compile-time or runtime behavior — an implementer must invent one. And
invent they will differently: zero-fill (Go), mask the count (C, Rust
release), trap (Rust debug). `memory-and-runtime.md` guarantees that what
this specification fixes, it fixes for every conforming program, with two
closed exception lists; an unspecified shift outcome on a construct the
language defines is a hole in that claim.

### Self-hosting needs two compilers to agree

Bootstrapping a compiler written in Xulo against the Rust implementation
requires identical observable behavior wherever the spec speaks. Shifts by
`>= width` are ordinary in bit-twiddling code (hash mixing, bitstream
packing); if one implementation masks and another traps, the same source
compiles and runs differently under each — a conformance divergence the
whole point of freezing v1 is meant to prevent.

### The cheap fix is already written elsewhere in the spec

Division by zero is exactly this problem, already solved: constants are
`E0210`, non-constants are a runtime trap
([`../../expressions/operators.md`](../../expressions/operators.md)).
Shift-amount-out-of-domain is the same class of operand-domain error and
takes the same rule and the same diagnostic code.

## Scope

**In scope.** The shift-amount domain rule (`0 … width−1`) with the
`E0210`/runtime-trap split; the "shifts never overflow" clarification; the
`E0210` trigger text; one row in each of the two identical runtime-failure
tables; the three other passages that enumerate `E0210` conditions
(`checking-rules.md`, `coercion.md`, `let-and-assignment.md`).

**Out of scope.** Wrapping/rotating arithmetic (a separate change); the type
rule for shift operands (unchanged: same integer type both sides); new
diagnostic codes; any change to bit-fill semantics, precedence, or `<< 0`.

## Compatibility

No conforming program can regress: the behavior of out-of-range shift counts
is currently unspecified, so no program has a defined meaning to lose.
Implementations that masked or zero-filled must trap instead — they were
already non-conforming under the closed-list claim if they did so silently.
In-range shifts, the overwhelmingly common case, are untouched.

## Non-goals

- No `wrappingShl`-style escape hatch here (see the wrapping change).
- No shift-amount type relaxation (`u64 << u8` stays `E0201`).

## Readiness

- [x] Depends on nothing unaccepted; every delta is written against the
      current, folded chapters.
- [x] All seven affected passages are covered by [`specs/`](specs/) and
      [`tasks.md`](tasks.md), including the two tables that MUST stay
      identical.

# Coercion and Implicit Conversion — Delta: Shift Count Domain

> **Delta** — replaces the constant-arithmetic sentence below the promotion
> table in `type-system/coercion.md`.

The sentence replaces the current one (the promotion table above it and the
paragraph below it are unchanged):

Constant division or remainder by zero, constant results that overflow the
type, and constant shifts by a negative or out-of-range amount are
compile-time errors (`E0210`); the same conditions with non-constant
operands are runtime traps.

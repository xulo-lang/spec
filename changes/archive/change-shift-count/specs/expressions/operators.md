# Operators — Delta: Shift Count Domain

> **Delta** — replaces the shift paragraph of the “Bitwise and Shift” section
> in `expressions/operators.md`.

## Bitwise and Shift

The shift paragraph replaces the current one (the paragraphs before and
after it are unchanged):

Both operands of a binary bitwise operator MUST be integers — `int` or a
fixed-bit integer type — and both MUST have the same type; the result has
that same type. `~` is the prefix complement of an integer and yields the
operand's type. Shifting is defined as follows: `<<` fills from the right
with `0`; `>>` fills from the left with the sign bit for signed types and
with `0` for unsigned types; bits shifted out are discarded. The shift
amount MUST be a non-negative integer strictly less than the bit width of
the operands' type — 64 for `int`, `W` for `usize` and `isize`, the type's
own width otherwise. An amount outside that range is a compile-time error
(`E0210`) when the value shifted and the amount are constants, and a
runtime trap otherwise. Inside the range a shift never overflows: the
bit-fill above is the result, however many bits leave. `x << 0` and
`x >> 0` are both the identity function.

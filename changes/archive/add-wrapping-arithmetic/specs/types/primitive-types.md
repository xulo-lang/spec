# Primitive Types — Delta: Overflow Cross-Reference

> **Delta** — appends one sentence to the “Overflow” bullet in
> `types/primitive-types.md`; the rest of the bullet and its neighbours
> are unchanged.

- **Overflow.** An integer operation (`int` or any fixed-bit integer type) that exceeds the range of its result type is a compile-time error (`E0210`) when every operand of the operation is a constant expression, and a runtime trap otherwise. A runtime trap is a panic: the program stops where the condition occurs, and there is no handler (see [`../memory-and-runtime.md`](../memory-and-runtime.md)). A literal that does not fit a fixed-bit type it is coerced into is likewise a compile-time error (`E0209`). These rules govern the operators; the `Math::wrapping*` intrinsics define wrapping results and never trap (see [`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md)).

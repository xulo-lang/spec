# Type Errors and Diagnostics — Delta: E0210 Shift Trigger

> **Delta** — replaces the `E0210` row of the “Type and expression errors”
> table in `type-system/errors.md`.

### Type and expression errors

The `E0210` row replaces the current one; every other row is unchanged:

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0210` | constant arithmetic divides by zero, takes a remainder by zero, overflows its type, or shifts by a negative or out-of-range amount | `const H = 1 / 0` | ``division by zero in constant expression`` | [operators.md](../expressions/operators.md) |

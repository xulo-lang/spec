# Checking Rules — Delta: Shift Count Domain

> **Delta** — replaces the `<< >>` row of the Operators table and the
> constant-evaluation sentence of the “Bindings” rule in
> `type-system/checking-rules.md`.

### Operators

The shift row replaces the current one; every other row is unchanged:

| Operators | Operands | Result |
|-----------|----------|--------|
| `& \| ^ ~ << >>` | integers of one and the same type (shift amount a non-negative integer below the type's width; a constant amount outside is `E0210`) | that type |

### Statement rules — Bindings

The constant-evaluation sentence replaces the current one (the rest of the
binding rule is unchanged):

Every `let` has an initializer; a `const` initializer MUST be a constant
expression, whose division or remainder by zero, out-of-range results, and
negative or out-of-range shift amounts are `E0210`.

# Variable Bindings and Assignment — Delta: Shift Count Domain

> **Delta** — replaces the constant-example block and the constant-evaluation
> bullet in the `const` section of `statements/let-and-assignment.md`.

The example block replaces the current one:

```xulo
const APP_NAME = "Xulo"
const MAX_COUNT = 100
const LIMIT: int = MAX_COUNT * 2

const A = 2
const B = A * 3            // legal: arithmetic over constants
const C = B > 5            // legal: comparison over constants

fn compute(): int { 42 }
const BAD = compute()      // error: not a constant expression
const ZERO = 1 / 0         // error: constant division by zero
const SHIFTED = 1 << 64    // error: constant shift amount out of range
```

The constant-evaluation bullet replaces the current one:

- Constant evaluation obeys the numeric error rules: constant division or
  remainder by zero, constant out-of-range results, and constant shifts by a
  negative or out-of-range amount are compile-time errors
  ([`../type-system/checking-rules.md`](../type-system/checking-rules.md)).

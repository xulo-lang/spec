# Calls — Delta: Const Generic Arguments

> **Delta** — replaces the opening paragraph of `## Generic calls` and the
> phrase about function-type callees in `expressions/calls.md`; the rest is
> unchanged.

Replace the opening paragraph with:

Type arguments of a generic function are inferred at the call site from the
actual arguments and from the expected type. A call MAY also write them
explicitly, between the callee and the argument list, and the written list
takes the place of inference for that call. A callee that declares const
parameters MUST be given a written list — `count<3>()` — because const
arguments are never inferred (`E0816`). The written list covers type and
const parameters together: its length is `E0808`, and a type/const kind
mismatch is `E0813`:

Extend the example that follows with one const-generic call:

```xulo
fn count<const N: int>(): int { N }

let c = count<3>()          // N is 3, a const argument is always written
```

Replace “a call through a value of function type has no type parameters and
accepts none” with:

a call through a value of function type has no generic parameters and
accepts none.

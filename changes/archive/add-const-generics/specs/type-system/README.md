# Type System — Delta: Const Generic Arguments

> **Delta** — replaces the `Generic bounds` and `Type applications` rows of the
> checking-areas table and the “Local inference” bullet in
> `type-system/README.md`; the rest is unchanged.

Replace the `Generic bounds` row with:

| Generic bounds | Every type argument — inferred at a call or written in a type application — satisfies its parameter's bounds; a const argument has no bound |

Replace the `Type applications` row with:

| Generic arguments | Every call or type application supplies exactly the declared number of generic arguments, type and const combined; a generic name is never bare, a type or const parameter is never applied, and a const argument is always written |

Replace the “Local inference” bullet with:

- **Local inference** — inside a function body, an unannotated `let`, a
  closure's omitted annotations, and the type arguments of a generic call are
  inferred from the surrounding evidence, and never across a signature; const
  arguments are never inferred and MUST be written
  ([`../types/type-relations.md`](../types/type-relations.md)).

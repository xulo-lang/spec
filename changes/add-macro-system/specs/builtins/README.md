# Built-ins — Delta: Compile-Time Macro Layer

> **Delta** — extends the two-layer table of `builtins/README.md` to three
> layers, adds the six compile-time chapters to the Index, and adds `ast` to
> the resolution order. The existing prelude and intrinsic rows are unchanged.

## Three layers

Replace the section heading and table with:

| Layer | Contains | Chapter |
|-------|----------|---------|
| **Prelude** | the built-in types, the built-in protocols `Display` and `Debug` with `Format`, the namespaces (including `ast`), and the built-in names — in scope in every module | [`prelude.md`](prelude.md) |
| **Intrinsics** | the compiler-provided functions `print`, `println`, and `str`, together with the members of `Math`, `Time`, and `Task`, each with a defined signature | [`intrinsic-functions.md`](intrinsic-functions.md) |
| **Compile-time macros** | the macro input/output values (`ast::`), compile-time execution, the context and sandbox rules, the quota, the macro diagnostics, and the expansion cache | [`execution.md`](execution.md), [`context.md`](context.md), [`sandbox.md`](sandbox.md), [`quota.md`](quota.md), [`errors.md`](errors.md), [`cache.md`](cache.md) |

The compile-time layer is not a runtime library. Its values are ordinary
values when user code or a `#Test` touches them, but macro execution happens
during compilation, under the sandbox and quota rules of its chapters.

## Index

Add these rows:

| File | Contents |
|------|----------|
| [`execution.md`](execution.md) | the observable execution contract: values, macro calls, traps, quota interaction, determinism |
| [`context.md`](context.md) | the six read-only context operations and the `ast` source API |
| [`sandbox.md`](sandbox.md) | forbidden resources, `ast::compile_date`, and `E1007` |
| [`quota.md`](quota.md) | the three quota layers, what is counted, and `E1001` |
| [`errors.md`](errors.md) | the macro diagnostic formats, primary-span targets, and span chains |
| [`cache.md`](cache.md) | structured-input and context hashing, invalidation, and cache transparency |

## Availability and resolution

Replace “every name of both layers” in the Availability section with “every
name of all three layers”, and replace step 5 of the resolution order with:

> 5. the prelude — the built-in type names and the intrinsics (`print`,
>    `println`, `str`, `Math`, `Time`, `Task`, `ast`).

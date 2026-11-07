# Macro Quota

A macro runs with a budget. The budget bounds execution steps, memory, and call
depth so that a buggy or malicious macro fails the build instead of hanging
it. The budget is observable; the execution strategy that consumes it is not.

## Three layers

```typescript
struct GlobalQuota {
  max_steps: int = 1000000;  // abstract execution steps
  max_memory: int = 16000000;       // bytes
  max_depth: int = 100;             // active function and macro calls
}
```

1. **Global default.** The values above. A compiler MAY choose larger
   defaults, but the default configuration MUST be finite and MUST be
   documented with the compiler.
2. **Package configuration.** A package MAY override any field through its
   build configuration (the manifest). The manifest format is outside this
   specification ([README.md](../README.md), § 1.1); the compiler consumes the
   three values and records the source as `package`.
3. **Macro override.** A `#[macro]` declaration MAY set any field:

   ```xulo
   #[macro(kind = "derive", target = Model, max_steps = 5000000)]
   fn derive_model(input: ast::DeriveInput): Result<…> { … }
   ```

The effective quota of one field is the first value present in this order:
macro, package, global default.

## Scope of a budget

A **top-level expansion** is one invocation site (`#[derive(T)]`, `#[name(…)]`, or
`name!(…)`) and everything it causes through an `ast::call_*` entry point.
The top-level macro's effective quota initializes the budget for that entire
tree. Nested macros draw from the same counters and the same call depth; a
nested macro's own quota does not reset the parent's budget.

This rule makes the cost of a call site predictable: the budget belongs to the
site, not to whichever macro happens to be innermost. If a helper macro is
expensive, the root macro raises `max_steps`; the helper's own override
applies only when the helper is invoked directly as a top-level expansion.

## Counting

- **Steps.** `max_steps` counts abstract execution steps. One unit is one step;
  no particular execution strategy is required. The implementation MUST
  document a deterministic mapping from source operations to steps. The mapping
  MUST advance the counter at least once for each evaluated subexpression,
  function or macro call, return, context query, AST construction, and list or
  map operation; fusing or specializing operations MUST NOT charge less than
  the work they replace. The counter is checked before the step; exceeding the
  limit stops before the offending step executes.
- **Memory.** The live bytes of values, including AST nodes, strings, lists,
  maps, and structs. The counter is checked on allocation; exceeding the limit
  stops before the allocation completes.
- **Depth.** The number of active function and macro calls, including the root
  macro. Reaching the limit stops the call that would exceed it.

The current usage and the limit are both reported in `E1001`, together with
the quota source.

## Failure

```text
error[E1001]: macro `derive_model` exceeded its step quota
  --> macros/model.xulo:42:8
   |
42 |   generate_field!(field)?
   |   ^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: used 1000003 of 1000000 steps
   = note: quota source: global default
   = note: raise the limit in the package configuration or in
           #[macro(kind = "derive", target = Model, max_steps = …)]
```

The primary span is the source operation being executed when the limit was
crossed inside the macro definition; the span chain reaches the invocation. A
quota failure is `ExecutionError::QuotaExceeded`
([type-system/errors.md](../type-system/errors.md)).

## Determinism

Quota accounting is deterministic for a fixed compiler version, input, and
context observations. The budget does not depend on host load, wall-clock time,
or scheduling. Two runs that expand the same macro with the same structured
input and context observations consume the same budget and either both succeed
or both fail at the same source operation.

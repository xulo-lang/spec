# Type Checking

Type checking is the static analysis that runs after a file has been parsed
into a syntax tree and its names have been resolved. It assigns a type to every
expression, verifies every rule that relates types to the constructs that use
them, and either accepts the program or reports diagnostics. This chapter maps
the checking layer; the rules themselves are in the three files listed in the
index below, and the relations they consume — equality, subtyping,
assignability, coercion — are defined in
[`../types/type-relations.md`](../types/type-relations.md).

## Inputs and outputs

The checker's input is a parsed, name-resolved module together with an
environment assembled from that module and everything it imports; its output is
one of exactly two results:

```text
input:   syntax tree + resolved names + module environment Γ
output:  acceptance  |  one or more diagnostics
```

- **Acceptance** means every judgment defined by
  [`checking-rules.md`](checking-rules.md) holds for the program: it is
  well-typed and may run.
- **Diagnostics** are the errors and warnings of
  [`errors.md`](errors.md). An error blocks execution of the program; a warning
  does not. Checking a module requires only the *signatures* of the modules it
  imports, never their bodies, so a program is checked file by file against the
  interfaces its imports expose.

Checking is deterministic and order-independent at file scope: because
module-level `fn`, `struct`, `enum`, `trait`, `type`, and `impl` declarations
forward-reference each other
([`../modules/source-files.md`](../modules/source-files.md)), the order in
which declarations are checked does not change the result.

## What is checked

| Area | Checked |
|------|---------|
| Declarations | Signatures are annotated, names are unique, aliases terminate, and every declared type name resolves |
| Expressions | Every expression has a type, operands and callees fit, literals are in range |
| Statements | Bindings initialize, assignments target mutable places, control transfers are in scope |
| Patterns | Each pattern fits its scrutinee, `match` is exhaustive, no arm is unreachable |
| Visibility | Only `pub` names cross module boundaries, and no public signature exposes a private type |
| Component declarations | `@State`, `@Store`, `@Effect`, and `@Environment` appear only at the top level of a component body, and children have an admissible type |
| Generic bounds | Every bound a call site instantiates is satisfied by the chosen type argument |
| `impl` completeness | Every trait method is implemented exactly once, with a matching signature |

Each row is expanded in [`checking-rules.md`](checking-rules.md); a rule that
fails produces the diagnostic named in [`errors.md`](errors.md).

## Where types come from

A type enters checking from one of four places, and the checker never guesses
beyond them:

- **Annotations** — parameter types, `let x: T`, return types, and field
  declarations. Module-level signatures are fully annotated; this is the
  inference boundary of [`../functions.md`](../functions.md).
- **Literal defaults** — an integer literal is `int`, a float literal is
  `float`, a string literal is `string`, `true` and `false` are `boolean`, and
  a template literal is `string`, unless an expected type adapts a literal
  ([`coercion.md`](coercion.md)).
- **Local inference** — inside a function body, an unannotated `let`, a
  closure's omitted annotations, and the type arguments of a generic call are
  inferred from the surrounding evidence, and never across a signature
  ([`../types/type-relations.md`](../types/type-relations.md)).
- **Expected types** — checking is bidirectional: a context that already knows
  what it wants pushes that type into literals, empty collections, closures,
  and generic calls before they are inferred bottom-up.

## Index

| File | Contents |
|------|----------|
| [checking-rules.md](checking-rules.md) | Notation and environment, literal typing, expression rules, pattern typing, statement rules, declaration rules, constraint solving, worked examples |
| [coercion.md](coercion.md) | The closed set of implicit conversions, what never converts, string interpolation, numeric promotion, equality |
| [errors.md](errors.md) | Diagnostic model, error code ranges, every error category with its message shape, warnings, and example diagnostics |

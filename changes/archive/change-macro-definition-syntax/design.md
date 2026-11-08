# Design: Macro Definition Syntax

This document records the alternatives for naming and declaring a procedural
macro, the design chosen, and the consequences for the built-in attribute
surface, naming, checking, and diagnostics. The prose that replaces chapter
text is in [specs/](specs/); the motivation is in [proposal.md](proposal.md).

## Goals and constraints

1. **One place for the kind.** The definition should state the macro kind once,
   not once in a `kind` key and again in the parameter type.
2. **One place for the name.** For attribute and function macros, the macro
   name is the implementation function's identifier. For derive macros, the
   name is the target written at the invocation site (`#[derive(Target)]`).
3. **Keep the signature checkable.** The compiler must know which input and
   output records the implementation takes before it checks the body.
4. **Keep the quota override.** `max_steps`, `max_memory`, and `max_depth`
   remain optional per-definition values.
5. **Read like the invocation.** `#[macro_derive(Serialize)]` should relate
   directly to `#[derive(Serialize)]`.

## The redundancy

The accepted form is:

```xulo
#[macro(kind = "derive", target = Serialize)]
fn derive_serialize(input: ast::DeriveInput): Result<ast::DeriveOutput, ast::MacroFailure> { … }

#[macro(kind = "attribute", name = "with_banner")]
fn banner_macro(input: ast::AttributeInput): Result<ast::AttributeOutput, ast::MacroFailure> { … }
```

`kind` fixes the parameter and return types, and the parameter type fixes the
kind. `target` duplicates the derive target that `#[derive(Serialize)]`
already writes. `name` duplicates the public spelling; for attribute and
function macros that spelling can be the function identifier.

## Alternatives

### Keep `kind` and `name` — rejected

The current form is explicit and uniform: one built-in attribute, one place
for quota keys, and a function identifier that is always independent of the
macro name. It survives as the baseline, but it pays for that uniformity with
two metadata values that the signature and the invocation already determine.
It is also the form that prompted this change: a user reading
`#[macro(kind = "attribute", name = "with_banner")] fn banner_macro(…)` sees
the same name twice and the kind twice.

### One `#[macro]`, infer the kind from the input type — rejected

A single attribute could omit `kind`:

```xulo
#[macro(target = Serialize)]
fn derive_serialize(input: ast::DeriveInput): Result<…> { … }

#[macro]
fn with_banner(input: ast::AttributeInput): Result<…> { … }
```

The checker infers the kind from `ast::DeriveInput`, `ast::AttributeInput`, or
`ast::FunctionInput`. This removes `kind`, but it makes the definition's
intent implicit and moves the kind into the signature. A wrong or aliased
parameter type changes or hides the kind, and diagnostics have to reconstruct
the intended kind before they can say “use `#[macro]` for a function macro.”
The attribute no longer states what is being defined.

### Three definition attributes — chosen

The attribute name carries the kind:

```xulo
#[macro]
fn sql(input: ast::FunctionInput): Result<ast::FunctionOutput, ast::MacroFailure> { … }

#[macro_attr]
fn with_banner(input: ast::AttributeInput): Result<ast::AttributeOutput, ast::MacroFailure> { … }

#[macro_derive(Serialize)]
fn derive_serialize(input: ast::DeriveInput): Result<ast::DeriveOutput, ast::MacroFailure> { … }
```

This keeps the kind explicit without a `kind` string, and it reads the way the
macro is invoked. It is the chosen design.

**Pro.** No `kind` metadata; no `name` metadata for attribute/function macros;
the derive target is the attribute argument; the signature still fixes the
input and output types; the three kinds remain visible in source.

**Con.** Three built-in attribute names instead of one. The implementation
function identifier is the macro name for `#[macro]` and `#[macro_attr]`, so
those functions cannot be named independently. `macro_attr` and `macro_derive`
join the built-in attribute namespace.

## Chosen definition rules

### Attributes and names

| Definition attribute | Macro name | Invocation |
|----------------------|------------|------------|
| `#[macro]` | the function identifier | `name!(…)` |
| `#[macro_attr]` | the function identifier | `#[name(…)]` |
| `#[macro_derive(Target)]` | `Target` | `#[derive(Target)]` |

For `#[macro]` and `#[macro_attr]`, the function value and the macro share one
spelling in two namespaces. `pub fn sql` with `#[macro]` exports the value
`sql` and the macro `sql`; `pub fn with_banner` with `#[macro_attr]` exports
the value `with_banner` and the attribute macro `with_banner`. For
`#[macro_derive(Serialize)]`, the function identifier is independent and is
usually `derive_serialize`; `pub` exports the function value under its
identifier and the derive macro under `Serialize`.

### Signatures

| Definition attribute | Parameter | Return |
|----------------------|-----------|--------|
| `#[macro]` | `ast::FunctionInput` | `Result<ast::FunctionOutput, ast::MacroFailure>` |
| `#[macro_attr]` | `ast::AttributeInput` | `Result<ast::AttributeOutput, ast::MacroFailure>` |
| `#[macro_derive(Target)]` | `ast::DeriveInput` | `Result<ast::DeriveOutput, ast::MacroFailure>` |

A definition still requires a module-level, non-`async`, non-`extern`,
non-generic function with exactly one parameter and no `where` clause. The
signature of the chosen attribute is fixed; a function with the wrong input or
output type is `E1010` with a note that names the matching attribute.

### Arguments and quota

`#[macro]` and `#[macro_attr]` take no positional arguments. `#[macro_derive]`
takes exactly one positional identifier, the target, and no other positional
argument. All three accept the optional named keys `max_steps`, `max_memory`,
and `max_depth`, whose values are non-negative integer literals. The keys are
the same three the accepted macro system already defines; only their carrier
changes from a single `#[macro]` attribute to the definition attribute.

```xulo
#[macro_derive(Model, max_steps = 5000000)]
fn derive_model(input: ast::DeriveInput): Result<…> { … }
```

The target of `#[macro_derive]` MUST be a legal identifier and MUST NOT be a
keyword or reserved word. It MUST NOT be a built-in attribute name (`derive`,
`test`, `macro`, `macro_attr`, `macro_derive`), and it MUST NOT be `Debug`,
`Display`, or `ToString`. The function identifier of `#[macro]` and
`#[macro_attr]` is already a legal identifier; it MUST NOT be a built-in
attribute name.

### Definition checks

Exactly one definition attribute MAY appear on a function. Multiple definition
attributes are `E1010`. A function carrying a definition attribute MUST NOT
also carry a user attribute macro; registration happens before expansion, so
the implementation would diverge from the checked function. The declaration
MUST NOT be inside a function body, and its body MUST NOT contain a direct
user macro invocation; macro-to-macro calls use `ast::call_*`.

The old form `#[macro(kind = …)]` is removed. `#[macro]` now takes no
positional arguments, so `#[macro(kind = "derive", target = Serialize)]` is a
malformed `#[macro]` definition (`E1010`). The note points at
`#[macro_derive(Serialize)]`; the analogous note for the other two kinds points
at `#[macro]` and `#[macro_attr]`.

### Duplicates

A macro name occupies the macro namespace. Two definitions with one macro name
are `E1010`, including two macros of different kinds with one name. For
`#[macro]` and `#[macro_attr]`, the macro name is the function identifier, so
two such functions with one identifier are also the ordinary
duplicate-declaration error at the function level; the macro rule does not
turn that into a new kind of collision. For `#[macro_derive]`, two functions
with different identifiers may target the same `Target`; that is `E1010`.

A macro name MAY share a spelling with a value or type. For `#[macro]` and
`#[macro_attr]`, the function value deliberately shares the spelling with the
macro; for `#[macro_derive(Target)]`, `Target` may equal a trait name.

### Diagnostics

- A definition attribute on a non-`fn` host is the ordinary attribute-host
  error `E0109`.
- A malformed definition attribute is `E1010`; this includes the removed
  `#[macro(kind = …)]` form, a missing or extra `#[macro_derive]` target, a
  wrong input or output type, a wrong quota key, a nested definition, multiple
  definition attributes, a user attribute macro on the same function, a
  duplicate macro name, and a body containing a direct user macro invocation.
- The macro kind is not inferred from the signature. The chosen attribute
  states it; a wrong signature is an error against that attribute, not a
  silent change of kind.

## Consequences

- The grammar does not change: `macro`, `macro_attr`, and `macro_derive` are
  attribute names (`macro` is the reserved word already accepted in that
  position; the other two are ordinary identifiers and parse as attribute
  names). `#[macro_derive(Target)]` uses the existing attribute-argument
  grammar.
- The AST does not gain a node: a definition is an ordinary `FnDeclaration`
  whose `attributes` contains one of the three built-in `AttributeNode`s.
- The built-in attribute list grows from `derive`, `test`, `macro` to
  `derive`, `test`, `macro`, `macro_attr`, `macro_derive`. The built-in host
  table gains two `fn`-only rows and the `#[macro]` row keeps its meaning for
  function macros.
- The macro registry stores the kind selected by the definition attribute and
  the name selected by the table above. Invocation resolution is unchanged.
- `E1010`'s trigger and primary span widen from “the `#[macro]` attribute” to
  “the macro definition attribute”; `E0109` still covers a non-`fn` host.
- Every example and every generic “`#[macro]` function” phrase in the affected
  chapters is rewritten to the new definition form or to “macro implementation
  function.”

## Impact

| File | Change |
|------|--------|
| [`types/traits.md`](../../types/traits.md) | built-in attribute list and host table; the Macro definitions subsection |
| [`type-system/checking-rules.md`](../../type-system/checking-rules.md) | definition pre-pass wording; definition section; duplicate and order rules |
| [`type-system/errors.md`](../../type-system/errors.md) | `E1010` trigger, example, and note |
| [`names.md`](../../names.md) | macro entity row, scope row, declaration-order bullet, resolution paragraph |
| [`modules/visibility.md`](../../modules/visibility.md) | `pub` on a macro implementation function; package-boundary bullets |
| [`modules/import-export.md`](../../modules/import-export.md) | export mapping row; collision rules; well-formedness rows |
| [`builtins/sandbox.md`](../../builtins/sandbox.md) | generic “`#[macro]` function” wording |
| [`builtins/quota.md`](../../builtins/quota.md) | quota override example and carrier wording |
| [`builtins/errors.md`](../../builtins/errors.md) | `E1010` primary span, example, and note |
| [`builtins/cache.md`](../../builtins/cache.md) | macro metadata invalidation wording |
| [`testing.md`](../../testing.md) | “`#[macro]` function” wording |
| [`expressions/calls.md`](../../expressions/calls.md) | quasi-quote example |
| [`ast.md`](../../ast.md) | `#[macro]` note and definition references |
| [`introduction.md`](../../introduction.md) | procedural-macro bullet wording |

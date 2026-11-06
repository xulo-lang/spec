# Add Macro System

**Status:** draft · **Date:** 2026-11-05 · **Change ID:** add-macro-system
**Affects:** `grammar.md`, `ast.md`, `introduction.md`,
`type-system/errors.md`, `type-system/checking-rules.md`,
`expressions/calls.md`, `testing.md`, `memory-and-runtime.md`,
`types/traits.md`, `names.md`, `modules/import-export.md`,
`modules/visibility.md`, `builtins/prelude.md`, `builtins/README.md`, and the
six new chapters `builtins/execution.md`, `builtins/context.md`,
`builtins/sandbox.md`, `builtins/quota.md`, `builtins/errors.md`, and
`builtins/cache.md`

## Summary

Xulo gains **procedural macros**: compile-time Xulo functions that receive a
structured description of source and return a structured AST fragment. There
are three kinds:

- **derive macros**, invoked by `#[derive(Name)]`, which produce additional
  items — typically an `impl`;
- **attribute macros**, invoked as an attribute — `#[route("/v1/users")]` — which
  replace the declaration they decorate and may add items beside it;
- **function macros**, invoked as `name!(…)` — `sql!("SELECT …")` — which
  produce an expression.

A macro is defined by putting `#[macro(…)]` on an ordinary module-level
function:

```xulo
#[macro(kind = "derive", target = Serialize)]
fn derive_serialize(
  input: ast::DeriveInput
): Result<ast::DeriveOutput, ast::MacroFailure> {
  let name = input.target.name
  return Result::Ok(ast::DeriveOutput {
    items: ast::quote! {
      impl Serialize for ${name} {
        fn serialize(self): Json { Json { entries: [] } }
      }
    }
  })
}
```

```xulo
#[macro(kind = "attribute", name = "route")]
fn route_attribute(
  input: ast::AttributeInput
): Result<ast::AttributeOutput, ast::MacroFailure> {
  let path = match input.args[0] {
    ast::Value::String(s) => s,
    _ => return Result::Err(ast::MacroFailure {
      message: "route path must be a string",
      span: ast::current_span()
    })
  }
  if path.slice(0, 3) != "/v1" {
    return Result::Err(ast::MacroFailure {
      message: "route path must start with /v1",
      span: ast::current_span()
    })
  }
  return Result::Ok(ast::AttributeOutput {
    item: input.target.node,
    extras: []
  })
}
```

```xulo
#[macro(kind = "function", name = "sql")]
fn sql_macro(
  input: ast::FunctionInput
): Result<ast::FunctionOutput, ast::MacroFailure> {
  let text = match input.args[0] {
    ast::Value::String(s) => s,
    _ => return Result::Err(ast::MacroFailure {
      message: "sql! expects a string literal",
      span: ast::current_span()
    })
  }
  return Result::Ok(ast::FunctionOutput {
    expr: ast::node("StringLiteral", { text: ast::Value::String(text) })
  })
}
```

The macro implementation is ordinary checked Xulo code. At expansion time the
compiler executes it in a sandbox with an execution-step, memory, and
call-depth quota. The generated AST joins the program before global name
resolution and type checking, so generated code is checked exactly like
written code. `ast::quote! { … }` is the quasi-quote: it parses its body as
source and substitutes `${expr}` holes. Macro diagnostics carry a span chain
from the generated code back through the macro call to the macro definition.

The change also defines the macro namespace and its interaction with `use`
and `pub`, the built-in `ast` namespace that holds the input/output types, and
the diagnostic family `E1001`–`E1012` for compile-time execution and
expansion.

## Motivation

### The language cannot be extended by its users

The current specification has no macro mechanism
([`introduction.md`](../../introduction.md), *No macro system*), and declaration
attributes are a closed checker-interpreted registry
([`types/traits.md`](../../types/traits.md)). A library author who needs
serialization, routing, schema generation, test markers beyond `#[test]`, or a
small embedded DSL must either ask for a compiler change or write the
boilerplate by hand at every use site. Both options stop the library, not the
language, from growing.

### The missing case is structural, not textual convenience

The motivating cases all inspect a declaration and produce code from it:

- `#[derive(Serialize)]` on `struct User { … }` should generate the
  `impl Serialize for User`;
- `#[route("/v1/users", "GET")]` on a function should generate the registration
  beside the function;
- `sql!("SELECT …")` should validate a literal and produce the query value.

Textual substitution is not enough for these; the macro needs the field names,
types, attributes, and spans of the declaration. A structured input and a
structured output are the honest interface, and they let the generated code
participate in ordinary name resolution and type checking.

### Procedural macros are already Xulo

The macro author writes `struct`, `if`, `Result`, and `?` — the language the
specification already defines. The only new surface is the `#[macro]`
declaration, the `ast::` namespace of input/output records, `ast::quote!`, and
the macro invocation forms. Xulo deliberately does **not** add a second
pattern/template language (a `macro_rules!`-style declarative macro); see
[Non-goals](#non-goals) and [design.md](design.md).

## Scope

### In scope

- **Definitions.** `#[macro(kind = …, …)]` on a module-level `fn`, with the
  three kinds, the fixed input and output types, and the definition checks.
- **Invocations.** `#[derive(Name)]`, user attributes `#[name(…)]`, and
  `name!(…)`; lookup in the macro namespace; import and export with `use`,
  `pub use`, and `pub`.
- **Structured inputs and outputs.** `ast::DeriveInput`,
  `ast::AttributeInput`, `ast::FunctionInput` and the corresponding outputs;
  `ast::AstNode`, `ast::Span`, `ast::Value`, `ast::MacroFailure`.
- **Quasi-quote.** `ast::quote! { … }` with `${…}` holes; hole values are
  strings, numbers, booleans, AST nodes, or lists of AST nodes.
- **Expansion.** The expansion loop before global name resolution; derive
  before attribute; same-kind source order; the 100-round limit; generated AST
  checked normally.
- **Compile-time execution.** The observable value model, context operations,
  sandbox, three-layer quota, expansion cache, and macro diagnostic format.
  The execution strategy and representation are implementation details.
- **Diagnostics.** The new family `E1001`–`E1012`, the span-chain rules, and
  the three error sources (macro author, execution, expansion loop).

### Out of scope

- **Declarative macros.** There is no `macro_rules!`-style pattern language.
  The rationale is in [design.md](design.md#declarative-macros) and
  [Non-goals](#non-goals).
- **A general-purpose `quote` keyword.** Quasi-quote exists only as
  `ast::quote!`, for macro bodies.
- **A public `TokenStream` interface.** `ast::quote!` parses source; the
  structured input/output types are the primary interface. A token-level
  view, if added later, is a lower-level escape hatch.
- **A macro package/registry protocol.** How macro packages are distributed
  and resolved is tooling; this change defines source-level import and
  visibility only.
- **Runtime code generation.** All expansion happens at compile time; there
  is no `eval` or runtime macro facility.
- **A general attribute DSL.** Attribute arguments are values, not arbitrary
  expressions; named and positional arguments are bounded by
  `AttributeArg`.
- **Changing the built-in `Debug` derive.** `#[derive(Debug)]` keeps its
  current structural expansion and its `E0112` generic restriction. User
  derive macros may handle generics because they receive the generic
  parameters and decide themselves.

## Compatibility

- **Purely additive at the invocation surface.** `name!(…)` and `#[name(…)]`
  are new syntax; a program that does not write a macro is read exactly as
  before. `#[derive(Debug)]` and `#[test]` keep their existing meanings.
- **The attribute registry becomes extensible.** Before this change an
  unknown attribute is `E0108` and there is no way to register one. After it,
  a user attribute macro in scope registers its name; an unknown name is
  still `E0108`. Built-in attribute names (`Derive`, `Test`, `Macro`) cannot
  be redefined (`E1010`).
- **Error codes do not collide.** `E0801`–`E0818` already belong to traits,
  impls, and generic bounds ([`type-system/errors.md`](../../type-system/errors.md)).
  The macro family is `E1001`–`E1012`. The proposal's draft numbers
  (`E0801`–`E0809`) are remapped; see [design.md](design.md#error-codes).
- **`Error` is not overloaded.** The prelude's `Error` is a one-field runtime
  payload ([`error-handling.md`](../../error-handling.md)). Macro failures are
  `ast::MacroFailure`, which additionally carries an optional `ast::Span`.
- **Existing macro-free code is unaffected.** A `#[macro]` function is also an
  ordinary function; marking it does not change any other declaration.
- **Implementation cost.** A conforming compiler needs a sandboxed execution
  engine, an expansion cache, and a merged AST pass before global name
  resolution. These are compile-time costs, not runtime costs.

## Non-goals

- **A declarative macro language.** Patterns and templates would be a second
  language to learn; the design goal is low user-side mental load, and
  procedural macros use the language already being learned.
- **Arbitrary compile-time I/O.** The sandbox forbids the filesystem,
  network, environment, process control, concurrency, wall-clock time, and randomness.
  `ast::compile_date()` is the single date-granular exception.
- **Macros as a runtime feature.** A macro expands once, at compile time;
  there is no runtime macro registry.
- **Hygiene by renaming.** Generated names are written by the macro author.
  The language does not rename identifiers; `ast::quote!` splices names
  exactly as written. A macro author who wants a fresh name writes one.
- **Replacing the closed built-in attributes.** `#[test]`, `#[derive(Debug)]`,
  and `#[macro]` stay built in; user macros extend the namespace beside them.
- **Changing the error-handling model.** A macro author reports a user error
  with `Result::Err(ast::MacroFailure { message: …, span: … })`; the checker reports it as
  `E1008`. Runtime panics during compile-time execution are `E1012`.

## Deviations from the supplied v0.3 draft

This change keeps the draft's semantics but adapts the surface to the
normative specification:

| Draft | This change | Why |
|-------|-------------|-----|
| `[derive(Serialize)]`, `#[macro(…)]` | `#[derive(Serialize)]`, `#[macro(…)]` | Xulo's attribute marker is `#[…]`, and `#[derive]` already exists |
| `[macro(derive, trait = Serialize)]` | `#[macro(kind = "derive", target = Serialize)]` | `derive`, `trait`, and `function` are keywords or reserved words; string kind values avoid them |
| `Error` in macro signatures | `ast::MacroFailure` | `Error` is already the prelude's runtime error payload with only `message` |
| `DeriveInput`, `AstNode`, … bare | `ast::DeriveInput`, `ast::AstNode`, … | keeps the macro surface in one namespace and adds one prelude name |
| `E0801`–`E0809` | `E1001`–`E1012` | `E0801`–`E0818` are already assigned to traits, impls, and generics |
| `ast::quote!({ … })` | `ast::quote! { … }` | a Xulo `{ … }` after a call is an ordinary block and cannot hold declarations; the brace-delimited quote is a distinct form |
| `var`/`$var` interpolation | `${expr}` holes | `${…}` already reads as “insert a value” in templates; `$name` stays available for quoted component code |
| `Error.span` | `ast::MacroFailure.span` | keeps span authorship without changing the prelude `Error` |

## Readiness

- [x] Alternatives, chosen design, and consequences — [design.md](design.md)
- [x] Grammar, AST, and invocation surface —
      [specs/grammar.md](specs/grammar.md), [specs/ast.md](specs/ast.md),
      [specs/expressions/calls.md](specs/expressions/calls.md)
- [x] Type and diagnostic rules —
      [specs/type-system/checking-rules.md](specs/type-system/checking-rules.md),
      [specs/type-system/errors.md](specs/type-system/errors.md)
- [x] Macro namespace, visibility, and imports —
      [specs/names.md](specs/names.md),
      [specs/modules/import-export.md](specs/modules/import-export.md),
      [specs/modules/visibility.md](specs/modules/visibility.md)
- [x] Built-in macro values —
      [specs/memory-and-runtime.md](specs/memory-and-runtime.md),
      [specs/builtins/prelude.md](specs/builtins/prelude.md)
- [x] Compile-time execution chapters —
      [specs/builtins/execution.md](specs/builtins/execution.md),
      [specs/builtins/context.md](specs/builtins/context.md),
      [specs/builtins/sandbox.md](specs/builtins/sandbox.md),
      [specs/builtins/quota.md](specs/builtins/quota.md),
      [specs/builtins/errors.md](specs/builtins/errors.md),
      [specs/builtins/cache.md](specs/builtins/cache.md)
- [x] Test story — [specs/testing.md](specs/testing.md)
- [x] Implementation and acceptance checklist — [tasks.md](tasks.md)

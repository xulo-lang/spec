# Design: Macro System

This document records the alternatives for Xulo's macro system, the design
chosen, and the consequences that choice carries: the surface syntax, the
structured interface, the expansion loop, compile-time execution, the macro
namespace, the diagnostics, and the places where the supplied v0.3 draft is
adapted to the normative specification. The prose that replaces chapter text
is in [specs/](specs/); the motivation and scope are in
[proposal.md](proposal.md).

## Goals and constraints

1. **Low user-side load.** `#Derive(Serialize)`, `#route("/v1/users")`, and
   `sql!("…")` must read like ordinary attributes and calls.
2. **Power for macro authors.** The author can inspect a declaration's fields,
   types, attributes, generics, and spans, and can return a structured AST.
3. **Generated code is ordinary code.** Expansion happens before global name
   resolution and type checking; generated nodes are checked like written
   nodes.
4. **Every failure is diagnosable.** User-authored failures, execution failures, and
   expansion-loop failures each have a code, a primary span, and a span chain.
5. **Xulo stays one language.** The macro surface uses existing tokens,
   existing attributes, and existing expression rules. No second pattern or
   template language is introduced.

## Declarative macros

**Decision: procedural macros only; no `macro_rules!`-style declarative
macros.**

- **Pro:** the macro author writes Xulo — `struct`, `if`, `Result`,
  `ast::quote!` — rather than a separate pattern/template language. The user
  learns one invocation form per kind: `#Derive`, `#attr`, `name!`. The
  structured interface is inspectable and type-checked.
- **Con:** simple substitution is more ceremony than a declarative macro. A
  one-line “wrap this expression” macro needs a function body.
- **Alternative rejected:** declarative macros. They would add a second
  grammar (patterns, fragments, repetition, hygiene) and a second error model,
  for a convenience the core use cases do not need. They also make expansion
  depend on textual token patterns, which conflicts with the rule that macro
  input is structured data.

**Consequence.** The language keeps the draft's “macro author is a library
author, not a compiler author” property, at the cost of no ultra-light
syntactic sugar.

## Attribute surface

**Decision: reuse Xulo's `#Name(args)` form; do not import Rust's `#[…]` or
`[derive(…)]` spelling.**

Xulo already has a declaration-attribute form (`#Derive(Debug)`, `#Test`)
with its own grammar, AST node, registry, and errors (`E0108`–`E0112`). The
draft's `[derive(Serialize)]` and `#[macro(…)]` would either duplicate that
form or replace it. Duplicating gives two attribute syntaxes and two
registries; replacing invalidates every existing attribute example and error.
Reusing `#` keeps one form.

The user-facing forms are therefore:

```xulo
#Derive(Serialize)
struct User { name: string, age: int }

#route("/v1/users", "GET")
fn get_users() { … }

let table = sql!("SELECT * FROM users WHERE id = id")
```

The definition form is `#Macro(kind = …, …)` on a module-level function. The
kind is a string (`"derive"`, `"attribute"`, `"function"`) because `derive`,
`trait`, and `function` are keywords or reserved words in Xulo
([`lexical-structure.md`](../../lexical-structure.md)). The target/name key
depends on the kind:

| Kind | Required keys | Macro name |
|------|---------------|------------|
| `"derive"` | `target = <Identifier>` | the trait name in `target` |
| `"attribute"` | `name = "<identifier>"` | the string |
| `"function"` | `name = "<identifier>"` | the string |

Optional keys `max_steps`, `max_memory`, and `max_depth` set the
macro-level quota ([builtins/quota.md](specs/builtins/quota.md)). The keys are written in the
macro's own `#Macro` attribute, for example
`#Macro(kind = "derive", target = Model, max_steps = 5000000)`.

### Attribute arguments

The current grammar admits only identifiers as attribute arguments:
`AttributeArg = Identifier`. That is enough for `#Derive(Debug)` but not for
`#route("/v1/users")` or `#Macro(kind = "derive", target = Serialize)`. The change
widens the production to a value with an optional name:

```text
AttributeArg   = [ Identifier "=" ] AttributeValue ;
AttributeValue = Identifier | StringLiteral | IntegerLiteral
               | FloatLiteral | BooleanLiteral ;
```

The registry entry decides the shape: `#Derive` still accepts only names,
`#Test` keeps its fixed string-title form, and `#Macro` accepts the fixed keys
above. A well-formed attribute with the wrong shape is the existing `E0110`;
a macro whose `#Macro` metadata is malformed is `E1010`. Extending the
attribute-value production is preferable to a macro-only second argument
syntax: it keeps “one attribute form” true and leaves a future built-in
attribute free to take a string or integer.

**Rejected:** a separate `#Macro` mini-language with its own brackets. It
would be a second attribute grammar, and the existing registry already has a
shape check.

## Definition: implementation functions

**Decision: a macro is an ordinary module-level `fn` carrying `#Macro`.**

- The function remains an ordinary function. It is type-checked normally and
  can be called and tested at runtime. At expansion time the compiler executes
  it in the sandbox.
- The macro is registered under the name in `#Macro`; the function's own
  identifier is independent. A derive macro's name is its target trait; an
  attribute or function macro's name is the `name` string.
- The function's `pub` controls the visibility of both the function value and
  the macro. A private macro cannot be reached from another module.
- `#Macro` is registered on `fn` only (another host is `E0109`). On a
  module-level, non-`async`, non-`extern`, non-generic function with exactly
  one parameter and a `Result<Output, ast::MacroFailure>` return type it is
  valid; anything else is `E1010`.

**Alternative considered: dedicated `macro fn` syntax.** Rejected: Xulo
already uses declaration attributes for modifier-like metadata, and a new
keyword would take a name from the keyword pool for no expressiveness gain.
**Alternative considered: compile-time-only functions with no runtime value.**
Rejected: it would make the implementation function untestable as an ordinary
function, and the structured input types are constructible values. The
sandbox is an execution mode, not a separate language.

### Input and output types

The draft leaves `ExpressionInfo`, `AttributeInfo`, `ParamInfo`,
`VariantInfo`, `GenericParam`, and `QualifiedMacroName` undefined, and
`ast::AttributeInput.target` carries no original node. The last omission is fatal:
an attribute macro must return `ast::AttributeOutput.item`, but without the original
node it cannot return the decorated declaration. This change defines:

- `ast::DeclarationInfo.node` — the original declaration AST node, so an
  attribute macro can return `input.target.node` unchanged or splice it into
  quoted code;
- `ast::ExpressionInfo.node` — the original expression node of a function
  macro call, so the macro can inspect or return it;
- the missing field records (`ast::FieldInfo`, `ast::VariantInfo`,
  `ast::ParamInfo`, `ast::GenericParam`, `ast::AttributeInfo`);
- `ast::AttributeInput` with positional `args` and a named `named_args` map,
  instead of a single map that cannot express `#route("/v1/users")`'s positional
  argument;
- namespaced type names under `ast::` so the macro surface adds exactly one
  prelude name, `ast`.

All definitions are in
[specs/memory-and-runtime.md](specs/memory-and-runtime.md#compile-time-macro-values).

**Alternative considered: bare `DeriveInput`, `AstNode`, `Value`.** Rejected:
ten new globally reserved names, including the very broad `Value`, for a
feature that many programs never use. The `ast` namespace keeps the surface
together and lets a program shadow `ast` in an inner scope like any other
built-in.

## Macro namespace and imports

**Decision: macro names live in a separate namespace from values and types.**

- The key of a macro is its fully qualified name: module path plus the
  macro's name (the `target` for a derive macro, the `name` string
  otherwise). Two modules may both define `w`; `m1::w` and `m2::w` are
  different macros. Short `use` names collide under the ordinary `E0104`
  rule.
- A plain `use m::w` imports the macro `w` as well as any value or type `w`;
  `use m::w as w2` aliases all three. `use type` imports no macro. A path
  that resolves only to a macro is a valid `use`.
- `pub use` re-exports macros; `pub` on the `#Macro` function exports them.
- A macro may share a spelling with a value or type in the same module; the
  namespaces are independent. Two macros with one name in one module are
  `E1010`.
- A macro name MUST be a legal identifier and MUST NOT be a keyword or a
  reserved word. It MUST NOT be a built-in attribute name (`Derive`, `Test`,
  `Macro`), and a derive macro MUST NOT also use a reserved derive target
  (`Debug`, `Display`, `ToString`).

**Alternative considered: one namespace for macros, values, and types.**
Rejected: `#Derive(Serialize)` needs the macro `Serialize` and the trait
`Serialize` to be usable under one spelling, exactly as `use serde::Serialize`
brings both into scope in the draft. A single namespace would force a rename
of one of them and make the common derive case awkward.

**Alternative considered: `use macro m::w`.** Rejected: it adds a keyword or
a contextual form to `use`; a separate namespace already makes the import
unambiguous.

## Expansion order

**Decision: derive first, then attributes; same-kind source order; function
macros in expression order; new macro calls in the next round.**

On one declaration:

1. every `#Derive` target in written order, left to right;
2. every attribute macro in written order, top to bottom.

Across the file, declarations are visited in source order. Function macro
calls are expanded when the containing expression is visited.

The reason for fixing source order is predictability: `#auth` followed by
`#route` is a way for the author to say that the authentication wrapper must
exist before the route macro runs. Leaving same-kind order to the
implementation makes cross-macro dependencies non-portable.

**Alternative considered: attributes first, then derive.** Rejected: an
attribute macro could remove or replace the declaration before the derive
runs, so a derive would no longer know what it decorates. Derive-first
matches the existing mental model (the type exists; the derive adds an impl).

**Alternative considered: iterate to a fixed point in one pass.** Rejected:
the round limit and the “new calls wait a round” rule make termination
checkable and diagnostics stable. A macro that generates a macro call is
normal; a macro that does so forever is `E1004`.

### Attribute macros cannot delete the decorated declaration

`ast::AttributeOutput.item` is mandatory and must be a declaration of the same
host kind as the input. A macro that wants to suppress the original code
returns the original node; a macro that wants to replace it returns another
declaration of the same kind. It cannot return `unit` or omit the item, and it
cannot turn a `struct` into a `fn`.

This rule exists because deleting a declaration after a derive has generated
an impl leaves an impl for a type that no longer exists. The failure would be
reported at the generated impl, not at the attribute that caused it — a
diagnostic that points away from the repair. The mandatory `item` removes the
possibility.

**Alternative considered: `item: Option<ast::AstNode>`.** Rejected for the reason
above; a “delete” attribute is a footgun in a language with derives.

## Local name resolution before expansion

**Decision: names in macro arguments are resolved against the pre-expansion
AST; an unresolved name is `E1009`, a hard error.**

Function macro arguments are expressions. The compiler checks that every name
they mention resolves before the macro runs. It does **not** evaluate the
expression; a literal argument becomes a scalar `ast::Value`, and any other
expression becomes an `ast::AstNode` value. The macro can inspect or quote the
node.

A name generated by an earlier macro is not in the pre-expansion AST, so a
later macro argument cannot refer to it. The repair is to hand the whole
region to one function macro, or to use an `ast::call_*` entry point from
inside the macro.
Deferring the resolution to after expansion would make the expansion order
depend on generated names, and the same source could expand differently in
two compilations; the cache key would become unstable.

Attribute arguments are bare `AttributeValue`s, not expressions. A bare
identifier argument is passed as its spelling (a string), not resolved as a
value. `E1009` therefore applies to function macro arguments only.

**Alternative considered: pass an unresolved token stream and let the macro
resolve.** Rejected: it moves type-checking into every macro, and a macro
failure would be reported at the macro author instead of at the misspelled
name in the user's source.

## Quasi-quote

**Decision: `ast::quote! { … }` is a brace-delimited token fragment with
`${expr}` holes; it returns `List<ast::AstNode>`.**

The body is not an ordinary Xulo block: a block cannot contain `impl`,
`struct`, or `fn`, and the motivating derive example is an `impl`. The
grammar therefore gives `ast::quote!` its own brace-delimited form, whose
pieces are balanced tokens and `${ Expression }` holes. The compiler
substitutes the holes and parses the result as declarations and statements.

Hole values are converted to tokens:

| Value | Inserted as |
|-------|-------------|
| `string` | the string's characters, lexed as source (usually an identifier or a type) |
| `int` / `float` | a numeric literal |
| `bool` | `true` / `false` |
| `ast::AstNode` | the node's own tokens |
| `List<ast::AstNode>` | the concatenation of the nodes' tokens |

A hole that does not form valid syntax after substitution is `E1011`. A
string is lexed as source rather than emitted as a string literal because the
common case is `${name}` for a declaration name or `${trait_name}` for a type;
to insert a string literal, a macro writes quotes inside the quoted body or
constructs the literal node explicitly.

**Alternative considered: automatic interpolation of every identifier that
happens to name a macro local** (the draft's `var` behaviour). Rejected: it
makes a quoted identifier silently change meaning when a local of the same
name appears, and there is no escape that is obvious to a reader. `${…}` is
explicit and leaves `$name` available for quoted component bindings.
**Alternative considered: `ast::quote!({ … })` exactly as drafted.**
Rejected: after `!`, `({ … })` is a parenthesized block expression; it cannot
hold the declarations the derive case needs. The brace form is a distinct
grammar form and reads like the quasi-quote it is.

## Compile-time execution

### Execution strategy

**Decision: the specification defines observable execution behaviour, not an
execution strategy.**

[builtins/execution.md](specs/builtins/execution.md) defines the values a
macro produces, its failures, the macro-call protocol, the quota bounds, and
determinism. The representation and execution strategy — an interpreter, a
compiler, or something else — are implementation details. They belong to the
implementation documentation, not this specification.

**Alternative considered: mandate one execution strategy.** Rejected: it
would make an internal representation normative, prevent other strategies, and
add no observable guarantee. The quota still needs a deterministic step
mapping, which any strategy can provide.

**Alternative considered: run macros through the ordinary runtime with a
capability object.** Rejected: the runtime has I/O, time, and unlimited
recursion; a macro must be interruptible on an execution-step budget. A
separate sandboxed execution mode makes the quota and sandbox enforceable
regardless of the strategy.

### Ordinary functions, sandboxed execution

A macro function is also an ordinary function. When the compiler invokes it,
execution is sandboxed: no filesystem, network, environment, process,
concurrency, randomness, or wall-clock access; `ast::compile_date()` is the
only date source. When user code invokes the same function at runtime, it is
an ordinary function call and the sandbox does not apply. A function that performs I/O can
therefore be tested at runtime but fails as a macro with `E1007`; this is
intentional and is stated in
[builtins/sandbox.md](specs/builtins/sandbox.md).

**Alternative considered: forbid runtime calls to macro functions.** Rejected:
it makes the implementation untestable with `#Test` and adds a second calling
convention for no safety gain.

### Context

**Decision: compile-time execution exposes read-only lookup of the program's
compile-time context.**

`ast::lookup_type`, `ast::lookup_function`, `ast::lookup_variable`,
`ast::fields_of`, `ast::current_span`, and `ast::span_of` read the
pre-expansion environment and the current invocation; they cannot mutate
declarations. The source signatures are in
[builtins/context.md](specs/builtins/context.md).

**Alternative considered: pass the whole environment as an input record.**
Rejected: it copies a large structure on every macro call and makes the
snapshot semantics ambiguous. The context operations read the same environment
the compiler already has.

### Macro-to-macro calls

**Decision: kind-specific entry points with typed outputs.**

`ast::call_derive`, `ast::call_attribute`, and `ast::call_function` take the
matching input record, or an `ast::Value` target node, and return the matching
output record. A target of another kind is `E1006`; an `ast::Value` that cannot
be converted to the input record is `E1002`. A typed input record is used as is.
This keeps every field of the input and output available on both sides of the
call; there is no untyped output bridge. A macro that needs another kind
constructs that kind's input record from its public fields.

### Quota

**Decision: three layers, with the macro layer winning.**

1. global defaults: 1,000,000 execution steps, 16,000,000 bytes, 100 active
   calls;
2. package configuration: tooling-supplied overrides (the manifest format is
   outside this specification);
3. macro `#Macro` keys.

A top-level macro expansion owns a budget initialized from the root macro's
effective quota; nested calls through an `ast::call_*` entry point draw from
the same budget and depth. The step and memory counters are per root expansion,
not per call, so a greedy child cannot reset the parent's budget. `E1001` reports
the quota source so the user knows which layer to change.

**Alternative considered: one flat global budget per module compilation.**
Rejected: one expensive macro would starve every other macro in the file, and
a macro author could not raise a limit locally.

### Cache

**Decision: cache by structured input, observed context, the macro
implementation, compiler version, and compile date.**

The key is `(qualified macro name, hash of structured input, hash of context
observations, hash of macro implementation, compiler version, compile date)`.
The structured input hash includes every span and every string in the input
record; the file's text and comments are not hashed as a whole. The context
hash records the query sequence and the answer to every lookup or span query,
including queries made by a macro reached through `ast::call_*`. The
implementation hash covers the definition of every such reached macro as well.
The execution strategy is not part of the key. Therefore an edit invalidates an
entry when it changes an observed span, a record string, a context answer, or
the macro code; an edit that leaves all of those unchanged may share. A failed
expansion is not cached. The cache is transparent; a miss and a hit must
produce the same diagnostics and AST.

**Alternative considered: no cache.** Rejected: derive macros run on every
type in a file, and a fixed-point expansion loop can invoke the same macro
many times. A cache keeps a large file's compile time proportional to the
distinct inputs, not the number of call sites.

## Error codes

The draft assigns `E0801`–`E0809` to macros. Those codes are already normative:

| Code | Existing meaning |
|------|------------------|
| `E0801` | a type argument satisfies no bound / no impl |
| `E0802` | an impl omits a trait method |
| `E0803` | an impl method has the wrong signature |
| `E0804` | an impl declares a method the trait does not |
| `E0805` | two impls of one trait for one type |
| `E0806` | a bound does not name a trait |
| `E0807` | generic arguments on a non-generic callee |
| `E0808` | wrong number of generic arguments |
| `E0809` | ambiguous member call |

Codes are stable and MUST NOT be reused
([`type-system/errors.md`](../../type-system/errors.md)). The macro family is
therefore `E1001`–`E1012`, a new range:

| Code | Owner | Meaning |
|------|-------|---------|
| `E1001` | execution | quota exceeded — steps, memory, or depth |
| `E1002` | execution | input value conversion failed |
| `E1003` | execution | recursive macro call |
| `E1004` | expansion | maximum rounds exceeded |
| `E1005` | expansion | macro not found |
| `E1006` | expansion | macro kind or signature mismatch |
| `E1007` | execution | sandbox violation |
| `E1008` | macro author | `ast::MacroFailure` returned |
| `E1009` | expansion | local name unresolved in a macro argument |
| `E1010` | expansion | invalid `#Macro` definition |
| `E1011` | expansion | malformed quote or unholeable value |
| `E1012` | execution | macro execution trapped (index, division, `panic`) |

The draft proposed reusing “the existing out-of-range code” for execution traps.
There is no such code: out-of-range indexing, division by zero, and `panic`
are **runtime failures**, not checker diagnostics, and carry no `E` code
([`memory-and-runtime.md`](../../memory-and-runtime.md)). At compile time the
failure is a compile-time trap rather than an ordinary runtime panic, so
`E1012` reports it and points into the macro definition with a span chain to
the invocation. This is a deliberate
new code, not a duplicate.

**Alternative considered: extend `E0801`–`E0818` into `E0819`–`E0830`.**
Rejected: the range is documented as “traits, impls, generic bounds”; mixing
compile-time expansion into it makes the family map misleading. `E1001` is a
new family with one owner.

### Three error sources

- `ast::MacroFailure` is authored by the macro. It means “the user's input is
  wrong”; the span falls on the attribute argument, derive target, or function
  macro argument the macro points at.
- `ExecutionError` is raised by compile-time execution. It means “the macro
  itself is buggy or over budget”; the span falls inside the macro definition.
- `ExpansionError` is raised by the expansion loop. It means registration,
  lookup, ordering, or round-count failure; the span falls on the invocation
  or definition that the loop can identify.

The separation is in the type system, not just the message: the compiler does
not have to guess whether a span came from the macro author or from the user.

## Consequences

- The grammar gains one attribute-value shape, one `#Macro` usage, one
  `name!(…)` call form, and one quote form. It gains no new token.
- The AST gains `MacroCallExpr`, `QuoteExpr`, `QuoteHole`, and a widened
  `AttributeArg`; `#Macro` is an ordinary `FnDeclaration` with an attribute.
- The attribute registry changes from “closed set” to “built-in set plus user
  attribute macros”. The built-in entries keep their positions and codes.
- The macro namespace is separate; `use`/`pub use` and `pub` include it. The
  module chapters gain the export and collision rules.
- The prelude gains one namespace, `ast`. The macro input/output records are
  members of it, not new reserved words.
- The checker gains a pre-resolution expansion pass and a set of macro
  invocation typing rules. Global name resolution and type checking run on the
  expanded AST exactly as before.
- The built-in chapter gains six compile-time chapters and a third layer in
  its index.

## Open questions

The following are intentionally left to later changes or to the
implementation, because they do not affect the surface this change defines:

1. **Execution strategy.** The chapter fixes observable behaviour, not the
   representation or the step mapping. These belong to the implementation
   documentation.
2. **Cache eviction and persistence.** The key and invalidation rules are
   fixed; whether the cache lives in memory, on disk, or both is an
   implementation choice.
3. **Macro debugging.** A macro author has spans and `panic`; a debugger for
   compile-time execution is tooling.
4. **Distribution of macro packages.** Macro visibility and import are
   defined; how a package ships macros is a manifest/tooling concern.
5. **`ast::call_*` at runtime.** It is specified to panic at runtime because
   the macro registry is a compile-time structure. A future change may expose
   a runtime registry or remove the function from runtime scope.
6. **Span fidelity through `ast::quote!`.** Generated nodes carry the quote
   call site unless a hole supplies a node with its own span. A later change
   may add per-fragment span annotations.

## Impact

| File | Gain |
|------|------|
| [`grammar.md`](../../grammar.md) | widened `AttributeArg`, attributes on `type`/`let`/`const`, `MacroCall`, `QuoteCall`, production index |
| [`ast.md`](../../ast.md) | `MacroCallExpr`, `QuoteExpr`, `AttributeArgNode`, note on `#Macro` as `FnDeclaration` |
| [`introduction.md`](../../introduction.md) | the macro non-goal becomes “no declarative macros; procedural macros are compile-time functions” |
| [`type-system/errors.md`](../../type-system/errors.md) | `E1001`–`E1012`, the family row, the three error-source enums |
| [`type-system/checking-rules.md`](../../type-system/checking-rules.md) | macro definition, invocation, expansion-order, and duplicate rules |
| [`expressions/calls.md`](../../expressions/calls.md) | macro call syntax, argument passing, `ast::quote!`, expansion typing |
| [`testing.md`](../../testing.md) | the three test layers for macros |
| [`memory-and-runtime.md`](../../memory-and-runtime.md) | `ast::` macro values and their fields |
| [`types/traits.md`](../../types/traits.md) | extensible attribute registry, `#Macro`, user derive targets, deletion rule |
| [`names.md`](../../names.md) | macro namespace and naming |
| [`modules/import-export.md`](../../modules/import-export.md) | macro export/import, collisions, unused use |
| [`modules/visibility.md`](../../modules/visibility.md) | `pub` macro visibility |
| [`builtins/prelude.md`](../../builtins/prelude.md) | the `ast` namespace |
| [`builtins/README.md`](../../builtins/README.md) | the compile-time layer and six chapter rows |
| `builtins/execution.md` | new chapter: observable execution behaviour, values, macro calls, determinism |
| `builtins/context.md` | new chapter: six read-only context operations and the `ast` source API |
| `builtins/sandbox.md` | new chapter: forbidden resources, `compile_date` |
| `builtins/quota.md` | new chapter: three layers, counting, `E1001` |
| `builtins/errors.md` | new chapter: diagnostic formats, span targets, span chains |
| `builtins/cache.md` | new chapter: structured-input and context hashing, invalidation, transparency |

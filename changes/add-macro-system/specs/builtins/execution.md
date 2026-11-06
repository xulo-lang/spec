# Macro Execution — Delta: New Chapter

> **Delta** — adds the new chapter `builtins/execution.md`; the body below is
> the whole post-acceptance chapter, heading included. The built-ins index row
> is in [`README.md`](README.md).

# Macro Execution

A macro implementation is an ordinary function executed while the program is
compiled. This chapter defines the observable execution contract: the values a
macro produces, when execution fails, how nested calls share a budget, and the
determinism the expansion cache relies on. It does not define an execution
strategy. A compiler MAY compile the function, interpret it, or use another
representation; the representation and internal details belong to the
implementation documentation.

## Observable contract

For a fixed macro implementation, input record, set of context observations,
effective quota, and compile date, one expansion produces exactly one of:

- the output record required by the entry point;
- an author-returned `ast::MacroFailure`, reported as `E1008`;
- a quota failure `E1001`;
- a sandbox violation `E1007`;
- a trap `E1012`;
- a macro-call failure (`E1002`, `E1003`, `E1005`, `E1006`), specified by the
  macro-call rules below;
- an expansion-loop failure (`E1004`, `E1009`–`E1011`), specified by
  [type-system/checking-rules.md](../type-system/checking-rules.md).

Internal values, temporaries, and evaluation strategy are not observable except
through these outcomes, the diagnostics, the quota usage, and the determinism
rule.

## Values

The public value model is `ast::Value`
([memory-and-runtime.md](../memory-and-runtime.md)):

```typescript
enum ast::Value {
  Unit, Bool(bool), Int(int), Float(float), String(string),
  List(List<ast::Value>), Map(Map<string, ast::Value>),
  AstNode(ast::AstNode),
}
```

A macro body may also hold ordinary user values — structs, enums, function
values, and closures — in locals. They obey the ordinary value semantics of
[memory-and-runtime.md](../memory-and-runtime.md), including ownership and
borrowing. They cannot cross the macro boundary: the boundary is the typed
input and output records and `ast::Value`. How a value is represented is not
observable and is not specified.

The memory quota ([quota.md](quota.md)) bounds live values, including AST
nodes, strings, lists, maps, and structs; the allocation and reclamation
strategy is an implementation choice.

## Execution

The implementation evaluates the macro body with the ordinary semantics of
Xulo functions. The sandbox rules ([sandbox.md](sandbox.md)) forbid filesystem,
network, environment, process, concurrency, wall-clock, and random access; the
only date source is `ast::compile_date()`.

A trap stops the current expansion and reports `E1012`. The traps are an
out-of-range list index, division or remainder by zero, and an explicit
`panic`. A trap does not unwind into user code; the macro either returns a
value or fails the compilation. The primary span is the source operation that
trapped in the macro definition, with a span chain to the invocation
([errors.md](errors.md)).

The call-depth quota bounds active function and macro calls. The root macro is
one active call; each ordinary function or macro call entered by the expansion
adds one. A call that would exceed the limit fails `E1001`
([quota.md](quota.md)).

## Macro calls

An `ast::call_*` entry point and a direct macro invocation use the same
observable call protocol:

1. Resolve the qualified name in the macro namespace. A missing name is
   `E1005`.
2. The callee's kind must match the entry point's kind (`ast::DeriveInput`,
   `ast::AttributeInput`, or `ast::FunctionInput`). A mismatch is `E1006`.
3. Use a typed input record as is. Convert an `ast::Value` input using the
   conversion table in [expressions/calls.md](../expressions/calls.md); a value
   that does not convert is `E1002`.
4. Run the callee on the same budget and the same call chain. A nested macro's
   own quota does not reset the parent's budget.
5. If the callee's qualified name is already active in the call chain, the call
   is `E1003`; the diagnostic lists the chain by fully qualified name.

For a direct invocation, the compiler constructs the same input record from the
invocation site. Malformed source is diagnosed by the ordinary checking rules,
not by a call failure.

A macro call is synchronous. There is no async, spawn, or concurrent macro
execution; compile-time execution is deterministic.

## AST values

An `ast::AstNode` carries:

- `kind` — the AST node kind string;
- `span` — the span it was created at; quote-created nodes carry the quote call
  site unless a hole supplied a node with its own span;
- immutable fields after construction. Rewriting a node means constructing a
  new one, which is what `ast::quote!` does.

The source-level operations `ast::node`, `ast::field`, and `ast::quote!` are
specified in [context.md](context.md) and
[expressions/calls.md](../expressions/calls.md). How the compiler represents
nodes or builds them is not specified. A node the compiler has inserted into
the program is not mutated.

## Determinism

For a fixed compiler version and implementation, a fixed input record, fixed
context observations, a fixed effective quota, and a fixed compile date, an
expansion produces the same output, the same diagnostics, and the same quota
usage. The execution environment has no random source, no wall clock, and no
concurrency.
`ast::compile_date()` is the only date input, at day granularity
([sandbox.md](sandbox.md)). The expansion cache relies on this rule
([cache.md](cache.md)).

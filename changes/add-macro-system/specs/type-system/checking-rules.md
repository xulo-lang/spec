# Checking Rules — Delta: Macro Rules

> **Delta** — adds a “Macros” section to `type-system/checking-rules.md` and
> states the expansion pass that precedes the ordinary checking rules. The
> existing literal, expression, statement, and declaration rules are
> unchanged; they apply to the expanded AST.

## Expansion pass

Macro expansion runs before global name resolution and before the ordinary
checking rules of this chapter:

1. Parse the file into an AST with spans.
2. Register every `#Macro` definition in the module's macro namespace; import
   macro names from compiled modules through `use`.
3. Repeat, up to 100 rounds:
   a. resolve names in function macro arguments against the current
      (pre-expansion) AST;
   b. expand derive macros, then attribute macros, in source order;
   c. expand function macro calls in expression order;
   d. insert generated nodes and collect any new macro calls.
4. Run global name resolution and the checking rules on the complete AST.

A new macro call produced by an expansion is expanded in the next round; a
macro call written inside another function macro's argument is expanded
before the outer call (it is part of the argument expression, not a product
of the outer expansion). Reaching the round limit is `E1004`; the primary
span is the invocation whose generated subtree was being expanded when the
limit was reached.

Within one declaration, a derive macro always receives the original
declaration, because derive expansion completes first. An attribute macro
receives the declaration as it stands after earlier attribute macros on that
declaration have run; the first attribute macro in source order sees the
written declaration. Context lookups still read the pre-expansion environment
([builtins/context.md](../builtins/context.md)).

## `#Macro` definitions

`#Macro` is a built-in attribute registered on `fn` only; on any other host it
is the ordinary attribute-host error `E0109`. On a `fn` it is valid only when
all of the following hold. A violation of this list is `E1010`. Macro
definitions are collected from the written AST before expansion; a `#Macro`
produced by an expansion is not registered and is `E1010`.

- It is not `async` and not `extern`, declares no generic parameters, and has
  no `where` clause.
- It has exactly one parameter, with no mode and no default.
- Its parameter type and return type match its kind:

| `kind` | Parameter | Return |
|--------|-----------|--------|
| `"derive"` | `ast::DeriveInput` | `Result<ast::DeriveOutput, ast::MacroFailure>` |
| `"attribute"` | `ast::AttributeInput` | `Result<ast::AttributeOutput, ast::MacroFailure>` |
| `"function"` | `ast::FunctionInput` | `Result<ast::FunctionOutput, ast::MacroFailure>` |

- For `"derive"`, `target` is required and names an `Identifier`; `name` is
  not written. For `"attribute"` and `"function"`, `name` is required, is a
  string holding a legal identifier, and is not a keyword or reserved word.
- `max_steps`, `max_memory`, and `max_depth`, when written, are
  non-negative integer literals.
- The macro name is not already registered in the module's macro namespace,
  is not a built-in attribute name (`Derive`, `Test`, `Macro`), and is not
  `Debug`, `Display`, or `ToString` for `"derive"`.
- The declaration is not inside a function body.

A `#Macro` function MUST NOT also carry a user attribute macro. Registration
happens before expansion, so such an attribute would replace the function
after the macro's implementation was registered; the registered implementation
and the checked function would diverge. The declaration is `E1010`. Other
attributes follow their own host rules.

## Invocation resolution

- **Derive.** For each target in `#Derive(T₁, …, Tₙ)`, left to right:
  `Debug` uses the built-in structural derive; `Display` and `ToString` are
  `E0111`; any other name resolves in the macro namespace as a `"derive"`
  macro. A missing name is `E1005`; a macro of another kind is `E1006`.
- **Attribute.** A built-in attribute name (`Derive`, `Test`, `Macro`) uses
  its built-in rule. Any other name resolves in the macro namespace as an
  `"attribute"` macro; a missing name is the ordinary `E0108`, and a macro of
  another kind is `E1006`.
- **Function.** The path of `name!(…)` or `m::name!(…)` resolves in the macro
  namespace as a `"function"` macro. A missing name is `E1005`; a macro of
  another kind is `E1006`.
- **Call entry.** An `ast::call_derive`, `ast::call_attribute`, or
  `ast::call_function` call applies the same kind check to its name; a target
  of another kind is `E1006`. A value that cannot be converted to the input
  record is `E1002`.

A macro name is not a value and not a type. Writing `let f = sql` or using a
macro name in type position is the ordinary unresolved-name error `E0101`;
the implementation function's own identifier is the only value name.

## Arguments

- A derive macro has no arguments beyond the decorated declaration.
- An attribute macro receives its positional `AttributeValue`s in `args` and
  its `name = value` entries in `named_args`. A bare identifier argument is
  the identifier's spelling, not a value lookup.
- A function macro receives its arguments as `ast::Value`s: a literal becomes
  a scalar value, and every other expression becomes an `ast::AstNode`.
  Before the macro runs, every name mentioned by an argument MUST resolve in
  the pre-expansion scope. A name that does not — including a name generated
  by an earlier macro — is `E1009`. The argument expression is not evaluated.

## Expansion results

- **Derive output.** `ast::DeriveOutput.items` are inserted immediately after the
  decorated declaration. Targets are processed left to right, and the items
  of the first target precede the items of the second. The derived impl
  participates in checking and dispatch exactly as a written impl.
- **Attribute output.** `ast::AttributeOutput.item` MUST be a declaration of the
  same host kind as the input (`fn`, `struct`, `enum`, `trait`, `impl`,
  `type`, `let`, or `const`); it replaces the original. `extras` are inserted
  immediately after it. A missing item, or an item of another host kind, is
  `E1006`. An attribute macro cannot delete its declaration.
- **Function output.** `ast::FunctionOutput.expr` replaces the `MacroCallExpr` in
  expression position. Its type is the type the ordinary rules give the
  expanded expression. An expansion that is not an expression is `E1006`.
- **Quote output.** `ast::quote!` yields `List<ast::AstNode>`. A hole value
  that cannot be inserted in its position — a non-scalar in an identifier
  position, a list where one node is required, or substituted text that does
  not parse — is `E1011`.

## Generated code

After the expansion pass, the checking rules of this chapter apply to the
expanded AST without special cases:

- Global name resolution sees every generated declaration.
- Type checking, borrow checking, mutability, exhaustiveness, and component
  rules apply to generated nodes.
- Every generated node carries the span of the macro invocation that produced
  it, unless a hole supplied a node with its own span. A diagnostic on
  generated code therefore points at the invocation and carries the macro
  definition in its span chain.
- A macro name is not in scope inside generated code unless it is imported or
  defined in the module that contains the invocation.

## Duplicate and order rules

- Two `#Macro` definitions with one macro name in one module are `E1010`.
- Two macros of different kinds with one name are also `E1010`: the macro
  namespace is one namespace.
- A macro may share a spelling with a value or type in the same module; those
  are different namespaces.
- Derive targets expand left to right; attribute macros expand top to bottom;
  derive expansion completes before attribute expansion on the same
  declaration. The order is observable and is part of the contract.
- A second invocation that would generate the same impl is the ordinary
  conflicting-impl error (`E0805`) when the generated impl is checked.

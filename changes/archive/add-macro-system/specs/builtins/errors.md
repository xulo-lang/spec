# Macro Errors — Delta: New Chapter

> **Delta** — adds the new chapter `builtins/errors.md`; the body below is the
> whole post-acceptance chapter, heading included. The codes themselves and
> the three error-source types are in
> [`type-system/errors.md`](../type-system/errors.md); the built-ins index row
> is in [`README.md`](README.md).

# Macro Errors

Every macro failure has a code, a primary span, and a span chain. The primary
span points at the code that is most useful to edit; the chain points back
through the invocation and the macro definition, so a reader can see both the
user's source and the macro's implementation.

## Model

A macro diagnostic uses the ordinary layout of
[type-system/errors.md](../type-system/errors.md):

- the header names the code and the category `semantic`;
- the primary span is the construct the message speaks about;
- secondary spans point at the invocation, the macro definition, and, for a
  nested call, each intermediate invocation;
- notes carry the repair and the quota or signature details.

Three sources own three types
([type-system/errors.md](../type-system/errors.md)):

| Source | Type | Means | Typical codes |
|--------|------|-------|---------------|
| macro author | `ast::MacroFailure` | the user's input is wrong | `E1008` |
| execution | `ExecutionError` | the macro is buggy or over budget | `E1001`, `E1002`, `E1003`, `E1007`, `E1012` |
| expansion loop | `ExpansionError` | registration, lookup, or rounds failed | `E1004`, `E1005`, `E1006`, `E1009`, `E1010`, `E1011` |

The compiler never reports a macro failure without one of these codes. A
runtime panic in ordinary code remains a panic without an `E` code.

## Primary-span table

| Code | Primary span | Who can repair |
|------|--------------|----------------|
| `E1001` | the source operation that crossed the quota, in the macro definition | the macro author, or the user who raises the quota |
| `E1002` | the `ast::call_*` call site in the macro | the macro author |
| `E1003` | the call that closes the cycle, in the macro definition | the macro author |
| `E1004` | the original invocation whose generated subtree kept expanding | the user or the macro author |
| `E1005` | the missing macro name in the invocation | the user (import or spelling) |
| `E1006` | the macro name in the invocation or the `ast::call_*` entry point | the user (wrong macro) or the macro author (wrong signature) |
| `E1007` | the forbidden operation in the macro definition | the macro author |
| `E1008` | `ast::MacroFailure.span` if present; otherwise the attribute argument, derive target, or function macro argument | the user |
| `E1009` | the unresolved name in the macro argument | the user |
| `E1010` | the `#[macro]` attribute or the offending key | the macro author |
| `E1011` | the quote hole or the substituted fragment | the macro author |
| `E1012` | the trapping source operation in the macro definition | the macro author |

## Span chain

For a direct invocation the chain is:

```text
primary (generated code or macro definition)
  → macro invocation site
  → macro definition site
```

For a macro that calls another macro:

```text
primary
  → inner macro definition site
  → inner invocation (the `ast::call_*` site)
  → outer macro definition site
  → outer invocation site
```

A generated node that carries a hole-supplied span starts the chain at that
span; otherwise it starts at the invocation. The chain never loops: the
recursion check (`E1003`) stops a cycle before the chain could repeat.

## Formats

### `E1001` quota

```text
error[E1001] semantic: macro `derive_model` exceeded its step quota
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

### `E1002` conversion

```text
error[E1002] semantic: macro input conversion failed
  --> macros/json.xulo:15:16
   |
15 |   ast::call_derive("m::derive", ast::Value::AstNode(target))?
   |   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: expected a struct or enum declaration
   = note: found `FnDeclaration`
   = note: `ast::call_derive` uses the node as `ast::DeriveInput`
```

### `E1003` recursion

```text
error[E1003] semantic: recursive macro call detected
  --> macros/a.xulo:15:8
   |
15 |   ast::call_attribute("m::b", input)?
   |   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: call chain: m::a → m::b → m::c → m::a
   = note: m::a defined at macros/a.xulo:3
   = note: m::b defined at macros/b.xulo:7
   = note: m::c defined at macros/c.xulo:12
```

### `E1004` rounds

```text
error[E1004] semantic: macro expansion exceeded maximum rounds (100)
  --> main.xulo:8:1
   |
 8 | recursive!(x)
   | ^^^^^^^^^^^^^
   |
   = note: last expanded: recursive! at round 100
   = note: expansion may be stuck in an infinite loop
```

### `E1005` not found

```text
error[E1005] semantic: cannot find macro `Serialize`
  --> main.xulo:3:9
   |
 3 | #[derive(Serialize)]
   |         ^^^^^^^^^
   |
   = note: no derive macro `Serialize` is in scope
   = note: import it with `use serde::Serialize;` or define it in this module
```

### `E1006` kind or signature mismatch

```text
error[E1006] semantic: `route` is a function macro, not an attribute macro
  --> main.xulo:3:2
   |
 3 | #[route("/v1/users")]
   |  ^^^^^
   |
   = note: function macros are invoked as `route!(…)`
```

The same code covers an `ast::call_*` entry point that names a macro of
another kind; its primary span is the entry point.

### `E1007` sandbox

```text
error[E1007] semantic: macro `sql` cannot access the filesystem
  --> macros/sql.xulo:12:3
   |
12 |   let text = File::read(path)
   |   ^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: compile-time macros may not access the filesystem, network,
           environment, processes, concurrency, wall-clock time, or randomness
```

### `E1008` macro-reported error

```text
error[E1008] semantic: macro `route` reported an error
  --> main.xulo:3:8
   |
 3 | #[route("/v2/users")]
   |         ^^^^^^^^^^^
   |
   = note: route path must start with "/v1"
   = note: reported by macros/route.xulo:9:5
```

`E1008`'s primary span is the error's own `span` when the macro supplies one.
Otherwise it is the attribute argument, the derive target, or the function
macro argument — the place the macro is talking about. The note carries the
author's message verbatim.

### `E1009` unresolved local name

```text
error[E1009] semantic: cannot resolve `user_name` in macro arguments
  --> main.xulo:8:14
   |
 8 | let q = sql!(user_name)
   |              ^^^^^^^^^
   |
   = note: macro arguments are resolved before expansion, so a macro cannot
           reference a name produced by another macro
```

### `E1010` invalid definition

```text
error[E1010] semantic: invalid macro definition: derive macro requires `target`
  --> macros/model.xulo:1:1
   |
 1 | #[macro(kind = "derive")]
   | ^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: write `#[macro(kind = "derive", target = Model)]`
```

### `E1011` malformed quote

```text
error[E1011] semantic: quote body is not well-formed
  --> macros/model.xulo:20:14
   |
20 |     items: ast::quote! { impl ${name} }
   |                          ^^^^^^^^^^^^^
   |
   = note: `${name}` inserted a field list where an identifier is required
```

### `E1012` trap

```text
error[E1012] semantic: macro `Serialize` trapped: index 99 out of range
  --> macros/serialize.xulo:17:11
   |
17 |   let field = fields[99]
   |               ^^^^^^^^^^
   |
   = note: list `fields` has 3 elements
```

## Relationship to the existing codes

- `E0108`–`E0112` keep their meanings for the attribute surface; a
  macro-specific definition failure is `E1010`.
- The macro system registers no codes in `E0801`–`E0818`; generated code still
  reports their ordinary errors, so a generated impl that collides with a
  written impl is `E0805` at the generated impl.
- Runtime failures — index, division, `panic` — have no `E` code in ordinary
  execution. During expansion they are `E1012`, because the compilation must
  stop with a diagnostic that points at the macro.

# Macro Cache

The expansion cache stores the result of a successful macro invocation so that
a large file does not re-run the same derive for every type. The cache is
transparent: a hit and a miss MUST produce the same AST and the same
diagnostics.

## Key

The key is a tuple:

```text
(qualified macro name,
 hash of structured input,
 hash of context observations,
 hash of macro implementation,
 compiler version,
 compile date)
```

- **Qualified macro name** — module path plus the macro name. Two modules may
  define `w`; `m1::w` and `m2::w` never share an entry.
- **Structured input** — a hash of the input record: node kinds, names,
  written types, field and variant order, attributes and their arguments,
  generic parameters, every span in the record, and every string in the
  record (including `ast::ExpressionInfo.text`). The file's source text and
  comments are not hashed as a whole. A whitespace or comment edit changes the
  key whenever it shifts an observed span or changes a record string; an edit
  that leaves every such value unchanged may share the entry.
- **Context observations** — the sequence of context queries the expansion
  performs and the result of each: `ast::lookup_type`, `ast::lookup_function`,
  `ast::lookup_variable`, `ast::fields_of`, `ast::current_span`, and
  `ast::span_of`, including those made by a macro reached through
  `ast::call_*`. An entry is served only when the same queries return equal
  values. A macro that reads no context has an empty observation set.
- **Macro implementation** — a hash of the macro's checked definition,
  including the bodies of functions it calls inside the module and the
  definition of every macro it reaches through `ast::call_*`. A change to the
  macro's code, or to a macro it calls, invalidates the entry even when the
  input is unchanged. The execution strategy is not part of the key.
- **Compiler version** — a compiler upgrade may change the AST, the execution
  semantics, or the span mapping.
- **Compile date** — the day returned by `ast::compile_date()`. It is always
  part of the key, so a macro that reads the date cannot serve yesterday's
  result.

The cache does not include the quota. A cached result was produced within the
quota that applied when it was created; a later invocation with a lower quota
MUST NOT use a cached result that would have exceeded that quota. The
implementation therefore records the step and memory usage with the entry and
checks it against the current budget before serving.

## Invalidation

An entry is invalid when any key component changes. In addition:

- a change to a module the macro imports invalidates macros whose
  implementation embeds that module;
- a change to the macro's `#[macro]` metadata — name, kind, target, quota — is an
  implementation change;
- a change to a context answer invalidates every entry that observed it;
- a compiler version change invalidates the whole cache;
- a failed expansion is not cached; the next compilation runs the macro again.

The cache is scoped to one compilation of a package tree. Within that
compilation, an entry may be reused across package boundaries — a macro defined
in a dependency and invoked in the current package — only when the full key
matches; a stale entry is never served.

## Transparency

The cache stores the output AST with its span mapping. On a hit the compiler
must produce the same generated nodes, the same spans, and the same
diagnostics as a miss. Because the key contains every span, record string, and
context answer the expansion can observe, a hit reproduces the same
evaluation; no span re-anchoring is required. A cache hit MUST NOT suppress a
diagnostic that a miss would produce, except that a successful expansion has
no diagnostic to suppress.

The cache is not a semantic boundary. A program's meaning does not depend on
whether the cache is enabled, warm, or empty. A compiler MAY disable the cache
without changing the language.

## Determinism

The cache relies on the determinism rule of compile-time execution: a fixed
implementation, fixed structured input, fixed context observations, and a
fixed compile date produce the same output ([execution.md](execution.md)). A
macro that reads a forbidden resource is `E1007`; a macro that reads
`ast::compile_date()` is deterministic within a day and correctly keyed by the
date.

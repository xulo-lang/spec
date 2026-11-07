# Prelude — Delta: The `ast` Namespace

> **Delta** — adds `ast` to the Built-in namespaces table, to the reserved-name
> list, and to the shadowing note of `builtins/prelude.md`. The rest of the
> prelude is unchanged.

## Built-in namespaces

Add this row to the table:

| Namespace | Contents | Specified in |
|-----------|----------|--------------|
| `ast` | the macro input/output records, `ast::AstNode`, `ast::Span`, `ast::Value`, `ast::MacroFailure`, the context functions, and `ast::quote!` | [`context.md`](context.md), [`execution.md`](execution.md) |

`ast` is the macro surface
([memory-and-runtime.md](../memory-and-runtime.md)). `ast::quote!` is the one
macro form in the namespace; `ast::call_derive`, `ast::call_attribute`, and
`ast::call_function` are the compile-time macro invocation helpers. Like
`Math`, `Time`, and `Task`, `ast` is a namespace name, not a value: it is
written only as the head of a `::` path. It enters resolution through the
prelude and MAY be shadowed in an inner scope exactly as any other
prelude name ([names.md](../names.md)).

## Reserved names

The Namespaces row of the reserved-name table becomes:

| Category | Names |
|----------|-------|
| Namespaces | `Math`, `Time`, `Task`, `ast` |

A module-level declaration named `ast` is `E0106`; an inner binding may shadow
it, after which the `ast::` types, the `ast::call_*` functions, and
`ast::quote!` are
unreachable in that scope until the shadow ends.

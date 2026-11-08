# Introduction — Delta: Macro Definition Attributes

> **Delta** — replaces the procedural-macro sentence of the “No declarative
> macro language” non-goal in `introduction.md`. The rest of the chapter is
> unchanged.

## Non-Goals

Replace the non-goal bullet with:

> - **No declarative macro language.** Xulo has no `macro_rules!`-style
>   pattern/template language and no token-substitution syntax. It has
>   **procedural macros** instead: a macro implementation function is ordinary
>   checked Xulo code that runs at compile time in a sandbox and returns a
>   structured AST fragment ([`builtins/execution.md`](builtins/execution.md)).
>   A macro is defined by `#[macro]`, `#[macro_attr]`, or
>   `#[macro_derive(Target)]` on a module-level `fn`. The invocation surface is
>   `#[derive(Target)]`, `#[name(…)]`, and `name!(…)`; the declaration-attribute
>   registry is the built-in set plus the attribute macros in scope, never an
>   untyped extension point
>   ([`types/traits.md`](types/traits.md)).

# Introduction — Delta: Macro Non-Goal

> **Delta** — replaces the “No macro system in v1” bullet in the non-goals
> list and the clause “there is no macro system” in the Swift/Rust comparison
> of `introduction.md`. The rest of the chapter is unchanged.

The non-goal bullet becomes:

> - **No declarative macro language.** Xulo has no `macro_rules!`-style
>   pattern/template language and no token-substitution syntax. It has
>   **procedural macros** instead: a `#Macro` function is ordinary checked
>   Xulo code that runs at compile time in a sandbox and returns a structured
>   AST fragment ([`builtins/execution.md`](builtins/execution.md)). The macro surface is
>   `#Derive(Name)`, `#name(…)`, and `name!(…)`; the declaration-attribute
>   registry is the built-in set plus the attribute macros in scope, never an
>   untyped extension point
>   ([`types/traits.md`](types/traits.md)).

The comparison clause becomes:

> borrows and modes are written with words (`p: mut T`), there are no
> declarative macros, and there are no classes to subclass.

The change keeps the original intent — no second pattern language and no
untyped syntax extension — while allowing a library author to extend the
language with checked, sandboxed compile-time code.

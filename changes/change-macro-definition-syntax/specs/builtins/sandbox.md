# Macro Sandbox — Delta: Macro Definition Attributes

> **Delta** — replaces the generic “`#[macro]` function” phrase in
> `builtins/sandbox.md`. The allowed/forbidden table, `ast::compile_date`, and
> failure rules are unchanged.

## Runtime versus compile time

Replace “The sandbox applies only when the compiler executes a `#[macro]`
function during expansion” with:

The sandbox applies only when the compiler executes a macro implementation
function during expansion.

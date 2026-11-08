# Tests — Delta: Macro Definition Attributes

> **Delta** — replaces the generic “`#[macro]` function” phrase in the macro
> implementation-test section of `testing.md`. The test layers, examples, and
> pass/fail rules are unchanged.

## Macros

Replace “A `#[macro]` function is an ordinary function, so a `#[test]` may
call its implementation identifier with constructed inputs:” with:

A macro implementation function is an ordinary function, so a `#[test]` may
call its implementation identifier with constructed inputs:

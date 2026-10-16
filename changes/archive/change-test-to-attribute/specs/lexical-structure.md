# Lexical Structure — Delta: `test` Leaves the Keyword Set

> **Delta** — edits the Keywords section in `lexical-structure.md`.

## Keywords

The count sentence becomes:

> The language has exactly the following closed set of **41** keywords; every
> other word is an ordinary identifier unless it is reserved below. No
> keyword MAY be used as one.

The `test` row is removed from the Declarations block of the keyword table;
every other row in the table is unchanged. The edited block reads:

| Category | Keyword | Purpose |
|----------|---------|---------|
| Declarations | `const` | Declares a compile-time constant |
| | `enum` | Declares an enumeration type |
| | `fn` | Declares a function or a function-typed value |
| | `mut` | Marks a binding or a borrow as mutable |
| | `self` | The receiver of a method inside an `impl` block; in a path, the current module |
| | `struct` | Declares a structure type |
| | `trait` | Declares a trait |
| | `type` | Declares a type alias |
| | `where` | Introduces a generic constraint clause after a signature |

`test` is **not** added to the reserved words: it becomes an ordinary
identifier — usable as a variable name, a function name, a field, anywhere an
identifier is allowed — exactly as `from` and `import` did when the import
syntax was retired. The old spelling `test fn …` no longer parses as a
declaration; it is a syntax error, never a reinterpretation.

# Lexical Structure — Delta: The Keywords `extern`, `usize`, and `isize`

> **Delta** — edits the **Keywords** section of `lexical-structure.md`.

The count sentence becomes:

> The language has exactly the following closed set of **61** keywords; every
> other word is an ordinary identifier unless it is reserved below. No keyword
> MAY be used as one.

Three rows join the table. `extern` enters the **Declarations** block,
between `enum` and `fn`:

| Category | Keyword | Purpose |
|----------|---------|---------|
| Declarations | `extern` | Declares foreign imports, or marks a function as exported under a foreign ABI |

`isize` and `usize` enter the **Primitive types** block, between `f64` and
`null`:

| Category | Keyword | Purpose |
|----------|---------|---------|
| Primitive types | `isize` | Signed pointer-width integer type |
| | `usize` | Unsigned pointer-width integer type |

Every other row is unchanged. `extern`, `isize`, and `usize` were ordinary
identifiers before this change: they appear in neither the keyword table nor
the reserved list, so a program that used one as a name is now ill-formed,
which is the ordinary consequence of a keyword addition
([`README.md`](README.md), § 2). The reserved-words list needs no edit —
by its own rule it excludes every keyword.

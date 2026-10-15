# Lexical Structure — Delta: Declaration Attributes

> **Delta** — replaces the reserved-symbol row of the token category table in
> `lexical-structure.md`, and the lexical-errors clause of the Conformance
> section.

The token category table gains `#` and loses the reserved-symbol row:

| Category | Tokens | Reading |
|----------|--------|---------|
| Grouping | `(` `)` `{` `}` `[` `]` | Call, block, list/map literal, subscript |
| Separators | `,` `;` | Item separator; optional statement terminator |
| Type and name separator | `:` `::` | Type/attribute label; enum variant path |
| Member access | `.` `?.` | Field, method, namespace member, tuple position; optional member |
| Assignment | `=` `:=` | Assignment; sugar for a mutable binding |
| Comparison | `==` `!=` `<` `>` `<=` `>=` | Equality and relational tests |
| Type test | `is` `not` | Type test and its negation |
| Range and spread | `..<` `...` | Half-open and closed ranges; spread inside a literal |
| Arithmetic | `+` `-` `*` `/` `%` `**` | Additive, multiplicative, modulo, power |
| Bitwise and shift | `&` `\|` `^` `~` `<<` `>>` | And, or, xor, complement, shifts |
| Logical and conditional | `!` `?` `??` `and` `or` | Negation, ternary `? :` and postfix propagation, nullish coalescing, connectives |
| Arrow | `=>` | Match arm body; closure body |
| UI and state | `@` `$` | State declarations; two-way binding prefix |
| Attributes | `#` | Declaration attribute marker |

The Conformance section's list of lexical errors loses its `#` clause:

> The lexical errors are: a character that cannot start any token (and a BOM
> away from the start), an unterminated block comment, string, or template,
> an invalid escape sequence, and an invalid number literal.

`#` now starts the attribute token; a `#` where no attribute is grammatical
fails in the parser, which carries the `syntax` category, not in the lexer.

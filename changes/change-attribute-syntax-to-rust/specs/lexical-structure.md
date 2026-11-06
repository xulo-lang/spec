# Lexical Structure — Delta: Rust-Style Attributes

> **Delta** — replaces the Attributes row of the operator and delimiter table
> in `lexical-structure.md` and adds the note that follows the table.

Replace the row:

| Category | Tokens | Reading |
|----------|--------|---------|
| Attributes | `#` | Declaration attribute opener; used only in `#[…]` |

`#` is still one token, but it is an attribute opener only as part of
`#[…]`. A `#` that is not followed by `[` is a syntax error in the parser,
not a lexical error: the lexer keeps recognizing `#`, and the attribute
productions simply do not match. The old `#Name` spelling is not an
attribute; `#Derive`, `#Test`, and `#Macro` all fail to parse. `[` and `]`
remain ordinary delimiters, so an attribute's brackets introduce no new token
class.

The built-in attribute names are lowercase (`derive`, `test`, `macro`); a
user attribute is an ordinary identifier. `derive` and `macro` remain
reserved words; they are accepted only in the attribute-name position and
are not identifiers there. `test` is an ordinary identifier, so nothing
special is needed for `#[test]`.

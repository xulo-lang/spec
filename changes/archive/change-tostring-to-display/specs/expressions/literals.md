# Literals — Delta: String Form — Display, Debug, and Format

> **Delta** — renames the protocol in the interpolation requirement paragraph
> of the "Template literals" section of `expressions/literals.md`. The
> requirement and the example are unchanged.

The paragraph becomes:

Every expression interpolated by `${}` MUST be a base type (`int`, `float`,
`bool`, `string`) or a type that implements the built-in `Display`
protocol; otherwise the program is ill-formed at compile time.

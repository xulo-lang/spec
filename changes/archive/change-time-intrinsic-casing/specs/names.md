# Names — Delta: Official Names Are Normative

> **Delta** — adds one paragraph after the RECOMMENDED preamble of the
> “Naming Conventions” section in `names.md`; the preamble, the table, and
> everything after it are unchanged.

## Naming Conventions

These conventions are RECOMMENDED, not enforced: breaking them produces no
diagnostic. The only name shapes the language itself rejects are keywords,
reserved words, and a lone `_` used as an ordinary identifier.

**Official names.** The conventions in the table are normative for every
name this specification defines itself — built-in types, protocols,
namespaces, prelude operations, and intrinsic functions: a built-in
function or method is always lowerCamelCase (`mapGet`, `nowNanos`), a
built-in type always UpperCamelCase (`List`, `Result`). The standard
library follows the same table; its chapters are documented outside this
specification ([`README.md`](README.md), § 1.1).

| Kind | Convention | Examples |
|------|------------|----------|
| Structs, enums, traits, type aliases, components | UpperCamelCase | `User`, `Theme`, `Area`, `VStack` |
| Enum variants | UpperCamelCase | `Theme::Dark`, `Level::High` |
| Functions, variables, parameters, fields, methods | lowerCamelCase | `sumTo`, `count`, `area`, `label` |
| Constants | UPPER_SNAKE_CASE | `PI`, `APP_NAME` |
| Type parameters | A single letter, never a descriptive word | `T`, `U`, `E`, `K`, `V` (a lowercase `t` or `e` is also a single letter) |
| Argument labels and attributes | lowerCamelCase | `spacing`, `variant`, `onClick` |

# Names — Delta: Snake-Case Naming

> **Delta** — replaces the “Argument labels” row of the “Identifiers and
> Named Entities” table and the “Naming Conventions” section in `names.md`.

## Identifiers and Named Entities (row)

The “Argument labels” row becomes:

| Argument labels | A named parameter or a component attribute | The label before `:` is the parameter's own name (`step_size`), or a component attribute declared by the component's contract (`onClick`); it is part of a signature, not a separate binding |

## Naming Conventions

These conventions are RECOMMENDED, not enforced: breaking them produces no
diagnostic. The only name shapes the language itself rejects are keywords,
reserved words, and a lone `_` used as an ordinary identifier.

A word boundary in a name is written `_`. The spelling lowercases every word
of the name and joins the words with a single underscore; an acronym is
spelled as its lowercase letters (`url`, `http`), and a digit stays attached
to the word it belongs to (`log2`, `atan2`). An ordinary name carries no
leading underscore, no trailing underscore, and no doubled underscore.

**Official names.** The conventions in the table are normative for every
name this specification defines itself — built-in types, protocols,
namespaces, built-in members, and intrinsic functions: a built-in function,
method, or property is always snake_case (`code_at`, `now_nanos`), a
built-in type always UpperCamelCase (`List`, `Result`). The standard
library follows the same table; its chapters are documented outside this
specification ([`README.md`](README.md), § 1.1).

| Kind | Convention | Examples |
|------|------------|----------|
| Structs, enums, traits, type aliases, components | UpperCamelCase | `User`, `Theme`, `Area`, `VStack` |
| Enum variants | UpperCamelCase | `Theme::Dark`, `Level::High` |
| Functions, variables, parameters, fields, methods | snake_case | `sum_to`, `count`, `area`, `label` |
| Constants | UPPER_SNAKE_CASE | `PI`, `APP_NAME` |
| Type parameters | A single letter, never a descriptive word | `T`, `U`, `E`, `K`, `V` (a lowercase `t` or `e` is also a single letter) |
| Const parameters | A single uppercase letter, never a descriptive word | `N`, `M` |
| Parameters declared as component attributes | lowerCamelCase | `spacing`, `variant`, `onClick` |

A parameter declared as a **component attribute** keeps lowerCamelCase: it
is a parameter of a component's API (`onClick`, `bgImage`, `spacing`), and
the component's contract, not the language, decides that it is an attribute.
It is still written and matched as an ordinary named argument; only its
spelling follows the component-attribute row. Every other named parameter
follows the function, variable, and parameter row: a named argument writes
the parameter's own name, so `splice`'s `delete_count` parameter is passed as
`delete_count: 2`.

Traits are named for the capability they describe, without an `I` prefix:
`Area`, not `IArea`; `Clone`, not `IClone`. Component names follow
UpperCamelCase, which is how a reader recognizes a component at a use site;
concrete component names such as `Text` and `Button` are ordinary imported
names, not reserved words. Names SHOULD be descriptive rather than abbreviated,
and SHOULD NOT repeat their kind in the name (`user_names` for a list of names,
not `user_names_list`).

A parameter declared as a component attribute keeps its lowerCamelCase
spelling even beside a snake_case function or field: its name is the
call-site vocabulary of the component API (`onClick`), and the component's
contract declares it as an attribute. Every other named parameter's label is
its parameter name and therefore follows the parameter rule. Types keep
UpperCamelCase so that an identifier beginning with a capital letter denotes
a type and an identifier beginning with a lowercase letter denotes a value or
a function.

# Traits — Delta: Test Title

> **Delta** — edits the opening paragraph of the **Attributes** section and
> replaces the **`### Test`** registry entry in `types/traits.md`.

## Attributes — opening paragraph, edited

The paragraph becomes:

A declaration MAY carry attributes written directly before it: the name
`#Name`, optionally followed by a parenthesized argument list; what the
arguments may contain is fixed per entry by the registry — `#Derive` takes a
comma-separated list of trait names with an optional trailing comma.

(The rest of the section — the example, the bullets, `### Derive` — is
unchanged.)

## Registry entry, replaced

### Test

`#Test` marks a function as a test. It is spelled only by the test form —
`#Test ("title")? [async] fn name() { … }` — whose shape is fixed
([`../testing.md`](../testing.md)):

- the declaration has no parameters, no return type, no generic parameters,
  no `where` clause, and no `pub`;
- the optional group holds one `StringLiteral` — the test's **title**,
  metadata for a tool's reporting, not a name; an empty group `()` and an
  absent group both mean no title, and the bare form is **RECOMMENDED** when
  no title is wanted;
- the identifier binds no value, and any reference to it is `E0107`.

Written on a `struct`, `enum`, `trait`, or `impl`, `#Test` is `E0109`
(`` attribute `Test` is not valid on a struct ``). A group that holds an
identifier, or a `fn` whose shape does not match the test form — parameters,
a return type, generics, a `where` clause, or `pub` — is `E0110`
(`` attribute `Test` takes the form `#Test ("…")? [async] fn name() { … }` ``);
a group holding anything else is a syntax error. When the sequence
`#Test` `(" StringLiteral? ")")? `async`? `fn` Identifier `(` `)` Block
appears at file scope, the declaration is a `TestDecl` and no registry
lookup runs ([`../grammar.md`](../grammar.md)).

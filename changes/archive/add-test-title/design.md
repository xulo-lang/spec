# Design: the test title

## The decision

```text
TestDecl = "#Test" [ "(" [ StringLiteral ] ")" ] [ "async" ] "fn"
           Identifier "(" ")" Block ;
```

The optional parenthesized group carries one payload: a `StringLiteral` —
the test's display title. An absent group and an empty group `()` both mean
no title, and the bare `#Test` is the canonical, **RECOMMENDED** spelling
when no title is wanted. The identifier remains the test's identity.

## Alternatives considered

### Title as an identifier — `#Test(parse_test)`

**Cost.** Reintroduces exactly the constraint the title exists to escape:
casing conventions forbid the spaces and punctuation a display phrase wants,
and an identifier-shaped title competes with the real identifier for a
reader's attention — two names, both looking like names. Rejected.

### Extend the generic `AttributeArgs` to accept literals

**Cost.** `AttributeArgs = AttributeArg { "," AttributeArg }` with
`AttributeArg = Identifier` is the grammar layer every registry entry shares.
Widening it to literals changes what *all* attributes may spell for the sake
of one entry, and leaves `#Derive` with a payload grammar it cannot use.
Entry-specific payloads belong in the entry's own production — which is
already how `#Test` works, since `TestDecl` carries the attribute name as a
literal. Rejected.

### Forbid the empty group — `#Test()` is `E0110`

**Cost.** The generic attribute form admits empty parentheses
(`[ "(" [ AttributeArgs ] ")" ]`), so a reader who types `#Test()` is
following the family's shape and receives an error for a no-op difference.
Rejecting it buys no disambiguation — the production's inner `StringLiteral`
is already optional — and spends a diagnostic on spelling preference.
Rejected; `()` is an accepted synonym.

### Make the title part of the identity

**Cost.** Identity is what `E0107`, discovery, and reporting already rest
on: the identifier binds nothing, two tests MAY share a name, and position
tells them apart. Folding the title into identity would make uniqueness a
question the language must answer — a tool property the spec excludes.
Rejected; the title is metadata, the identifier is the identity.

## Consequences

### Grammar and ambiguity

- `#Test("x") fn …` matches only `TestDecl`: the generic `Attribute`
  production cannot swallow a string payload, so the ordinary-`fn` path never
  sees it.
- `#Test(ident) fn …` does not match `TestDecl` (the group wants a string),
  parses as an ordinary `FnDecl` carrying the attribute, and is `E0110`.
- `#Test(123) fn …` matches neither — a syntax error, since `123` is
  neither a `StringLiteral` nor an `AttributeArg`.
- The disambiguation rule is unchanged in kind: when the token sequence
  `#Test` `(" StringLiteral? ")")? `async`? `fn` Identifier `(` `)` Block
  appears at file scope, the declaration is a `TestDecl` and no registry
  lookup runs.

### Semantics

- The title binds no value and is not a name in any namespace; referencing
  is not even syntactically possible — it is a string literal in metadata
  position.
- The identifier keeps every rule it has: identity as a tool reports it,
  no uniqueness requirement, any reference `E0107`.
- Uniqueness of titles is not required; a tool distinguishes same-titled or
  same-named tests by position, as today.

### AST

`TestDeclaration` gains `title: StringLiteral | null` — `null` exactly when
the group is absent or empty. It still carries no `attributes` field: the
title is a dedicated field, not an `AttributeNode` list.

### Diagnostics

| Code | Change |
|------|--------|
| `E0107` | unchanged — the identifier rule does not mention the title |
| `E0109` | unchanged — placement is the same |
| `E0110` | the test-form instance grows the optional group: `` attribute `Test` takes the form `#Test ("…")? [async] fn name() { … }` `` |

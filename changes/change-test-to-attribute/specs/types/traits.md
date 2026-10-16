# Traits — Delta: `#Test` Registry Entry

> **Delta** — edits the **Attributes** section of `types/traits.md` as it
> reads after [add-declaration-attributes](../../../add-declaration-attributes/specs/traits.md)
> is folded: extends the placement bullet and adds a `### Test` registry
> entry after `### Derive`.

## Placement bullet, edited

The bullet in the Attributes section becomes:

- A registered attribute is valid only on the declarations its registry entry
  allows (`E0109`), and its arguments must fit the shape that entry states
  (`E0110`). Attributes are written on the five item declarations — `fn`,
  `struct`, `enum`, `trait`, and `impl` — and on the test form (`#Test`), and
  on nothing else
  ([`../grammar.md`](../grammar.md)).

## Registry entry, added

### Test

`#Test` marks a function as a test. It is spelled only by the test form —
`#Test [async] fn name() { … }` — whose shape is fixed
([`../testing.md`](../testing.md)):

- the declaration has no parameters, no return type, no generic parameters,
  no `where` clause, and no `pub`;
- the identifier binds no value, and any reference to it is `E0107`.

Written on a `struct`, `enum`, `trait`, or `impl`, `#Test` is `E0109`
(`` attribute `Test` is not valid on a struct ``). Written before a `fn`
whose shape does not match the test form — parameters, a return type,
generics, a `where` clause, or `pub` — it is `E0110`
(`` attribute `Test` takes the form `#Test [async] fn name() { … }` ``).
When the sequence `#Test` `async`? `fn` Identifier `(` `)` Block appears at
file scope, the declaration is a `TestDecl` and no registry lookup runs
([`../grammar.md`](../grammar.md)).

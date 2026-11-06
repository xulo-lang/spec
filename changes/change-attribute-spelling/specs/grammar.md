# Grammar — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the `Attribute` production, adds `AttributeName` to
> the productions and the declarations index row, replaces the `TestDecl`
> production and its prose, and replaces the visibility sentence in the
> Declarations section of `grammar.md`. The `AttributeArgs`, `AttributeArg`,
> and `AttributeList` productions are unchanged.

## Attribute productions

Replace the `Attribute` production and add `AttributeName`:

```text
Attribute     = "#" "[" AttributeName [ "(" [ AttributeArgs ] ")" ] "]" ;
AttributeName = Identifier | "derive" | "macro" ;
AttributeArgs = AttributeArg { "," AttributeArg } [ "," ] ;
AttributeArg  = Identifier ;
AttributeList = { Attribute } ;
```

An attribute is written `#[name]` or `#[name(args)]`. The built-in attribute
names are lowercase. `derive` and `macro` are reserved words accepted only in the
attribute-name position; `test` is an ordinary identifier. Attribute names
occupy their own namespace and bind nothing. The argument list keeps its
existing shape; a change that widens `AttributeArg` for macro metadata is a
separate change. The declarations that carry an `AttributeList` are
unchanged.

The declarations row of the production index becomes:

| Nonterminal | Section |
|-------------|---------|
| `AttributeName`, `Attribute`, `AttributeArgs`, `AttributeArg`, `AttributeList`, `FnDecl`, `AbiTag`, `ExternBlock`, `ExternItem`, `ExternParamList`, `ExternParam`, `ExternFnDecl`, `ParameterList`, `Parameter`, `Receiver`, `StructDecl`, `FieldList`, `Field`, `EnumDecl`, `VariantList`, `Variant`, `VariantPayload`, `NameTypeList`, `NamedPayload`, `TraitDecl`, `TraitMethods`, `TraitMethod`, `ImplDecl`, `TypeAlias`, `LetDecl`, `MutableBinding`, `SimpleBinding`, `DestructuringBinding`, `IdentifierList`, `ConstDecl`, `TestDecl` | [Declarations](#declarations) |

Replace the visibility sentence in the Declarations section with:

Every alternative of `Declaration` MAY carry `pub`, except `impl` and
`TestDecl`: `impl` introduces no name and is never part of an interface, and
a `#[test]` declaration is never part of an interface either
([visibility.md](modules/visibility.md)); the four component declarations
carry no visibility marker either ([state.md](components/state.md)).

## Test declaration

Replace the `TestDecl` production with:

```text
TestDecl = "#" "[" "test" "]" [ "async" ] "fn" Identifier "(" ")" Block ;
```

Replace the `TestDecl` prose with:

A `TestDecl` is a function declaration with a fixed shape: `#[test]` precedes
`async` and `fn` (never `async #[test] fn`), the
parameter list and the return type are absent — the body is an implicit
`fn(): unit` — and it carries no `pub`, no generic parameters, and no `where`
clause. The identifier is the test's identity as a tool reports it, not a
name in the value namespace: referencing it is a compile-time error (`E0107`).
The optional `async` marker makes the body an `async` context
([async-expressions.md](expressions/async-expressions.md)); the
declaration's checking, inertness, and pass/fail criterion are specified in
[testing.md](testing.md).

The head is written exactly `#[test]`, with no argument group. The production
carries the attribute name as a literal, so the test form accepts no other
attributes. When the token sequence `#[test]` followed by `async`? `fn`
Identifier `(` `)` Block appears at file scope, the declaration is a
`TestDecl` and no attribute lookup runs.

When the full test form does not match, `#[test]` and `#[test(...)]` are
ordinary attributes on the `FnDecl`. A bare `#[test]` before a `fn` with
parameters, a return type, generic parameters, a `where` clause, or `pub` is
`E0110`; so are `#[test()]` and `#[test(name, …)]`, because the test attribute
takes no arguments. A `#[test(...)]` group that the current `AttributeArg`
grammar cannot parse — notably a `StringLiteral` or a numeral — is a syntax
error. When the `add-macro-system` change widens `AttributeArg`, such a group
parses and falls under the same `E0110` rule. The registry rules of
[traits.md](types/traits.md) give the message for the `E0110` case
(`` attribute `test` takes no arguments and is written `#[test]`, followed by
`[async] fn name() { … }` ``).
Written on a `struct`, `enum`, `trait`, or `impl`, `#[test]` is `E0109`.

The old forms `#Test`, `#Test("…")`, and `#Test()` no longer parse as an
attribute, and `#[Test]` is not a registered name. A `#` that does not begin
`#[…]` is a syntax error ([lexical-structure.md](lexical-structure.md)).

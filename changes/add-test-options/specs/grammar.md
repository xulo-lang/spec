# Grammar — Delta: Test Options

> **Delta** — replaces the `TestDecl` production, adds `TestOptions`,
> `TestOption`, and `TestExpectation`, replaces the `TestDecl` prose and the
> `#[test(...)]` fallback paragraph, and adds the new productions to the
> declarations production index row of `grammar.md`. `Attribute`,
> `AttributeArgs`, `AttributeArg`, `AttributeValue`, and `AttributeList` are
> unchanged: the accepted `add-macro-system` change already supplies the
> widened attribute-value shape.

## Test production

Replace `TestDecl` with:

```text
TestDecl = "#" "[" "test" [ "(" TestOptions ")" ] "]" [ "async" ] "fn"
           Identifier "(" ")" Block ;
TestOptions = TestOption { "," TestOption } [ "," ] ;
TestOption = TestExpectation | "skip" [ "=" StringLiteral ]
           | "timeout" "=" IntegerLiteral ;
TestExpectation = "should_fail" | "should_panic" [ "=" StringLiteral ] ;
```

The declarations row of the production index becomes:

| Nonterminal | Section |
|-------------|---------|
| `AttributeName`, `Attribute`, `AttributeArgs`, `AttributeArg`, `AttributeValue`, `AttributeList`, `FnDecl`, `AbiTag`, `ExternBlock`, `ExternItem`, `ExternParamList`, `ExternParam`, `ExternFnDecl`, `ParameterList`, `Parameter`, `Receiver`, `StructDecl`, `FieldList`, `Field`, `EnumDecl`, `VariantList`, `Variant`, `VariantPayload`, `NameTypeList`, `NamedPayload`, `TraitDecl`, `TraitMethods`, `TraitMethod`, `ImplDecl`, `TypeAlias`, `LetDecl`, `MutableBinding`, `SimpleBinding`, `DestructuringBinding`, `IdentifierList`, `ConstDecl`, `TestDecl`, `TestOptions`, `TestOption`, `TestExpectation` | [Declarations](#declarations) |

## Test declaration

Replace the `TestDecl` prose with:

A `TestDecl` is a function declaration with a fixed shape: `#[test]` or
`#[test(options)]` precedes `async` and `fn` (never `async #[test] fn`), the
parameter list and the return type are absent — the body is an implicit
`fn(): unit` — and it carries no `pub`, no generic parameters, and no `where`
clause. The identifier is the test's identity as a tool reports it, not a
name in the value namespace: referencing it is a compile-time error (`E0107`).
The optional `async` marker makes the body an `async` context
([async-expressions.md](expressions/async-expressions.md)); the declaration's
checking, inertness, options, and pass/fail criterion are specified in
[testing.md](testing.md).

The optional group is a non-empty comma-separated list of test options; a
trailing comma is allowed and the options may appear in any order. The
options in this change are:

- `should_fail`;
- `should_panic`, optionally followed by `= "text"`;
- `skip`, optionally followed by `= "reason"`;
- `timeout = N`, where `N` is an integer literal greater than zero.

At most one expectation and at most one `timeout` may appear; `skip` MUST be
the only option in its group. A group with neither an expectation nor `skip`
is a normal test; a group with only `timeout` is a normal test with a
deadline. A group with an expectation and `timeout` is valid in either order. The
`skip` and `timeout` rules, including the `timedOut` outcome, are in
[testing.md](testing.md).

The deferred option names `category`, `focus`, `retry`, and `threads` are not
`TestOption` productions in this change. They fall back to an ordinary
attribute when their group parses and produce `E0110`.

When the full test form does not match, `#[test]` and `#[test(...)]` are
ordinary attributes on the `FnDecl`. A marker before a `fn` with parameters, a
return type, generic parameters, a `where` clause, or `pub` is `E0110`; so is
a group that parses as an ordinary attribute but violates the option rules —
a duplicate or mixed option, `skip` combined with another option, an invalid
`timeout`, or an unknown option name. A `#[test(...)]` group that is not a
valid `TestOptions` group parses as an ordinary attribute and is `E0110`; a
positional value is therefore `E0110`, not a syntax error. The registry
rules of [traits.md](types/traits.md) give the message for the `E0110` case
(`` attribute `test` takes `should_fail`, `should_panic`,
`should_panic = "…"`, `skip`, `skip = "…"`, or `timeout = N` ``).

Written on a `struct`, `enum`, `trait`, or `impl`, `#[test]` is `E0109`.

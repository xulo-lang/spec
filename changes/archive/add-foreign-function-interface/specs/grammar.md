# Grammar — Delta: Extern Declarations

> **Delta** — edits the **Keywords** and **Program and modules** sections,
> the **Declarations** section, and the production index of `grammar.md`.

## Keywords

The count sentence becomes:

> The language has exactly the following closed set of 61 keywords; every other
> word is an ordinary identifier unless it is reserved.

The `Keyword` production gains `"extern"`, `"isize"`, and `"usize"`, in
alphabetical position:

```text
Keyword = "and" | "as" | "async" | "await" | "bool" | "break" | "const"
        | "continue" | "copy" | "defer" | "else" | "enum" | "extern" | "f32"
        | "f64" | "false" | "float" | "fn" | "for" | "i8" | "i16" | "i32"
        | "i64" | "if" | "impl" | "in" | "int" | "is" | "isize" | "let"
        | "lock" | "match" | "mod" | "move" | "mut" | "not" | "number" | "null"
        | "or" | "panic" | "pub" | "return" | "self" | "shared" | "spawn"
        | "string" | "struct" | "super" | "trait" | "true" | "type" | "u8"
        | "u16" | "u32" | "u64" | "unit" | "unknown" | "use" | "usize" | "where"
        | "while" ;
```

## Program and modules

`Declaration` gains the two extern forms:

```text
Declaration  = FnDecl | ExternFnDecl | ExternBlock | StructDecl | EnumDecl
             | TraitDecl | ImplDecl | TypeAlias | LetDecl | ConstDecl
             | TestDecl ;
```

The ambiguity note's parenthetical becomes: (`pub use` stays in the header;
`pub extern`, `pub fn` and the rest start the body). No other production
contains `extern`.

## Declarations

The section's opening prose is replaced in full:

Every alternative of `Declaration` MAY carry `pub`, except `impl` and
`TestDecl`: `impl` introduces no name and is never part of an interface, and
a `#Test` declaration is never part of an interface either
([visibility.md](modules/visibility.md)); the four component declarations
carry no visibility marker either
([state.md](components/state.md)). An `ExternFnDecl` is the exception in the
other direction: it MUST carry `pub`, since linkage with no export is not
expressible, and an `ExternBlock`'s `pub` exports every item of the block.
`async` follows `pub` and precedes `fn`; neither extern form carries it, and
neither appears in `Statement` — both are declarations of a module body only.
The five item declarations — `FnDecl`, `StructDecl`, `EnumDecl`, `TraitDecl`,
and `ImplDecl` — MAY carry an attribute list, written outermost, before `pub`
and `async`; which attributes are defined, and on which of them, is a
checking rule and not a grammatical one
([traits.md](types/traits.md)); the two extern forms carry no attribute list.
Generic parameters are declared once,
directly after the name, by functions,
`struct`s, `enum`s, type aliases, and `impl` blocks — never by a `trait`,
whose `GenericParams?` exists only so that an attempt can be diagnosed
(`E0810`)
([generics.md](types/generics.md), [traits.md](types/traits.md)).

Six productions are added after `FnDecl`:

```text
AbiTag          = StringLiteral ;
ExternBlock     = [ "pub" ] "extern" AbiTag "{" { ExternItem } "}" ;
ExternItem      = "fn" Identifier "(" [ ExternParamList ] ")" [ ":" Type ]
                  [ "as" Identifier ] ";" ;
ExternParamList = ExternParam { "," ExternParam } [ "," ] ;
ExternParam     = Identifier ":" Type ;
ExternFnDecl    = "pub" "extern" AbiTag "fn" Identifier
                  "(" [ ExternParamList ] ")" [ ":" Type ] Block ;
```

`ExternParam` is the parameter form without a mode and without a default,
which is why neither exists in an extern signature; `ExternFnDecl` reuses it
for the same reason, and its missing `GenericParams?`, `WhereClause?`,
`async`, and `AttributeList` positions — present on `FnDecl` — are what make
those markers syntax errors on an export. The prose accompanying the block
is:

- `AbiTag` admits any string literal; which literals are registered is a
  checking rule, not a grammatical one ([`ffi.md`](ffi.md)).
- An `ExternItem` has no `Block`: an import declares a signature and
  terminates in `;`, exactly as a `TraitMethod` does.

The production index's Declarations row becomes:

```text
| `Attribute`, `AttributeArgs`, `AttributeArg`, `AttributeList`, `FnDecl`, `AbiTag`, `ExternBlock`, `ExternItem`, `ExternParamList`, `ExternParam`, `ExternFnDecl`, `ParameterList`, `Parameter`, `Receiver`, `StructDecl`, `FieldList`, `Field`, `EnumDecl`, `VariantList`, `Variant`, `VariantPayload`, `NameTypeList`, `NamedPayload`, `TraitDecl`, `TraitMethods`, `TraitMethod`, `ImplDecl`, `TypeAlias`, `LetDecl`, `MutableBinding`, `SimpleBinding`, `DestructuringBinding`, `IdentifierList`, `ConstDecl`, `TestDecl` | [Declarations](#declarations) |
```

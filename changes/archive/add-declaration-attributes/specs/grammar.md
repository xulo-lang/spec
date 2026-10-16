# Grammar — Delta: Declaration Attributes

> **Delta** — replaces the `SymbolOperator` production, the opening prose and
> the declaration productions of the Declarations section, and the
> declarations row of the production index in `grammar.md`.

`#` joins the symbol inventory:

```text
Operator = WordOperator | SymbolOperator ;
WordOperator = "and" | "or" | "is" | "not" ;
SymbolOperator =
      "." | "?" | "!" | "=" | "==" | "!=" | "<" | ">" | "<=" | ">="
    | "+" | "-" | "*" | "/" | "%" | "**" | "<<" | ">>" | "&" | "|" | "^" | "~"
    | "=>" | "..<" | "..." | "?." | "??" | "@" | "$" | "::" | ":=" | "#" ;
Delimiter = "(" | ")" | "{" | "}" | "[" | "]" | "," | ":" | ";" ;
```

The Declarations section opens with the attribute rule and carries the
attribute list on the five item forms:

Every alternative of `Declaration` MAY carry `pub`, except `impl` and
`TestDecl`: `impl` introduces no name and is never part of an interface, and
a `test` is never part of an interface either
([visibility.md](modules/visibility.md)); the four component declarations
carry no visibility marker either
([state.md](components/state.md)). `async` follows `pub` and precedes `fn`.
The five item declarations — `FnDecl`, `StructDecl`, `EnumDecl`, `TraitDecl`,
and `ImplDecl` — MAY carry an attribute list, written outermost, before `pub`
and `async`; which attributes are defined, and on which of them, is a
checking rule and not a grammatical one
([traits.md](types/traits.md)). Generic parameters are declared once,
directly after the name, by functions, `struct`s, `enum`s, type aliases, and
`impl` blocks — never by a `trait`, whose `GenericParams?` exists only so
that an attempt can be diagnosed (`E0810`)
([generics.md](types/generics.md), [traits.md](types/traits.md)).

```text
Attribute     = "#" Identifier [ "(" [ AttributeArgs ] ")" ] ;
AttributeArgs = AttributeArg { "," AttributeArg } [ "," ] ;
AttributeArg  = Identifier ;
AttributeList = { Attribute } ;

FnDecl = [ AttributeList ] [ "pub" ] [ "async" ] "fn" Identifier
         GenericParams? "(" [ ParameterList ] ")" [ ":" Type ]
         WhereClause? Block ;

ParameterList = Parameter { "," Parameter } [ "," ] ;
Parameter     = Receiver | Identifier ":" [ "mut" | "shared" ] Type
                              [ "=" Expression ] ;
Receiver      = "self" | "mut" "self" ;

StructDecl = [ AttributeList ] [ "pub" ] "struct" Identifier GenericParams?
             "{" [ FieldList ] "}" ;
FieldList  = Field { [ "," ] Field } [ "," ] ;
Field      = [ "pub" ] Identifier ":" Type ;

EnumDecl    = [ AttributeList ] [ "pub" ] "enum" Identifier GenericParams?
              "{" [ VariantList ] "}" ;
VariantList = Variant { [ "," ] Variant } [ "," ] ;
Variant     = Identifier [ VariantPayload ] ;
VariantPayload = "(" ( TypeList | NameTypeList ) ")" ;
NameTypeList   = NamedPayload { "," NamedPayload } ;
NamedPayload   = Identifier ":" Type ;

TraitDecl    = [ AttributeList ] [ "pub" ] "trait" Identifier GenericParams?
               "{" [ TraitMethods ] "}" ;
TraitMethods = TraitMethod { ";" TraitMethod } [ ";" ] ;
TraitMethod  = [ "async" ] "fn" Identifier "(" [ ParameterList ] ")" ":" Type ;

ImplDecl  = [ AttributeList ] "impl" GenericParams? [ Identifier "for" ]
            Type "{" { FnDecl } "}" ;
TypeAlias = [ "pub" ] "type" Identifier GenericParams? "=" Type ;
```

A parameter MUST be annotated; `mut T` and `shared T` are parameter modes, not
types, whether the parameter belongs to a declaration or to a function type
(`FnTypeParam`) ([function-types.md](types/function-types.md),
[concurrency.md](concurrency.md)). A `,` between struct fields and between
variants is optional and a trailing `,` is allowed; a variant's payload slots
are all positional or all named, never mixed ([enums.md](types/enums.md)).
Inside a `trait`, signatures are separated by `;` with the final `;` optional,
and an attribute list on a `trait` method is not grammatical — the list
belongs to declarations, and `TraitMethod` is a signature, not a declaration.

The production index's declarations row gains the attribute productions:

| Nonterminal | Section |
|-------------|---------|
| `Attribute`, `AttributeArgs`, `AttributeArg`, `AttributeList`, `FnDecl`, `ParameterList`, `Parameter`, `Receiver`, `StructDecl`, `FieldList`, `Field`, `EnumDecl`, `VariantList`, `Variant`, `VariantPayload`, `NameTypeList`, `NamedPayload`, `TraitDecl`, `TraitMethods`, `TraitMethod`, `ImplDecl`, `TypeAlias`, `LetDecl`, `MutableBinding`, `SimpleBinding`, `DestructuringBinding`, `IdentifierList`, `ConstDecl`, `TestDecl` | [Declarations](#declarations) |

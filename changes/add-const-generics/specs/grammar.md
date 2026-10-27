# Grammar — Delta: Const Generic Parameters

> **Delta** — replaces the `GenericParam` production and adds the const
> parameter note after the generic productions in `grammar.md`; `TypeArgs`,
> `TypeList`, and the rest of the grammar are unchanged.

```text
GenericParams  = "<" GenericParam { "," GenericParam } [ "," ] ">" ;
GenericParam   = Identifier [ ":" TraitBound ]
               | "const" Identifier ":" Type ;
TraitBound     = Type { "+" Type } ;
WhereClause    = "where" WhereItem { "," WhereItem } ;
WhereItem      = Identifier ":" TraitBound ;
```

A const parameter's `Type` MUST be `int`; any other type is a compile-time
error (`E0812`, [`types/generics.md`](types/generics.md)). The type-argument
list is unchanged: a const argument is written in `TypeArgs` and interpreted
by the callee's parameter kind — an integer literal, a module `const` name, or
a const parameter name at a const position. A kind mismatch is `E0813`.

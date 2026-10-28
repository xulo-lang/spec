# Grammar — Delta: Const Generic Parameters

> **Delta** — replaces the `GenericParam` production and adds the const
> parameter note after the generic productions in `grammar.md`; `TypeArgs`,
> `CallTypeArgs`, `TypeList`, and the rest of the grammar are unchanged.

```text
GenericParams  = "<" GenericParam { "," GenericParam } [ "," ] ">" ;
GenericParam   = Identifier [ ":" TraitBound ]
               | "const" Identifier ":" Type ;
TraitBound     = Type { "+" Type } ;
WhereClause    = "where" WhereItem { "," WhereItem } ;
WhereItem      = Identifier ":" TraitBound ;
```

A const parameter's `Type` MUST be `int`; any other type is a compile-time
error (`E0812`, [`types/generics.md`](types/generics.md)). Both
`TypeArgs` (named-type applications) and `CallTypeArgs` (call applications)
keep their existing shape: a const argument is written in their shared
`TypeList` and interpreted by the callee's parameter kind — an integer
literal, a `const` binding, or a const parameter name at a const position. A
kind mismatch is `E0813`.

Because an integer literal has no sign, `-1` is a unary-minus expression, not
a const argument; a negative value is written through a `const` binding. An
integer literal has no type meaning outside a const position; literal types
the language otherwise defines remain usable at a type position. A call to a
callee with const parameters that omits the list is `E0816`, and an `impl`
const parameter that does not occur in the implemented type is `E0817`.

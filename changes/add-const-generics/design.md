# Design: const parameters of type int

## The decision

A generic parameter list may contain **const parameters**:

```xulo
fn count<const N: int>(): int { N }
fn repeat<const N: int>(value: int): int { N * value }
struct Buffer<const N: int, T> { head: T }   // N is available to the body/fields

count<3>()                 // 3
repeat<2>(5)               // 10
```

| Feature | Rule |
|---------|------|
| Declaration | `const N: int` in a generic parameter list; `int` is the only admitted type |
| Argument | integer literal, module `const` name, or const parameter name; written explicitly |
| Use | N is a compile-time `int` value in the declaration's body; it is never a type |
| Application | a declaration with const parameters requires the full generic argument list |
| Arithmetic | not in type arguments in version 1; ordinary arithmetic on N in a body is allowed |
| Inference | none for const parameters |

A type parameter is still a type and never a value; a const parameter is the
converse. They share one parameter list and one namespace of names, and a
duplicate name is an error.

## Kind-directed arguments

The grammar keeps the existing type-argument list:

```text
TypeArgs = "<" TypeList ">" ;
TypeList = Type { "," Type } ;
```

`count<3>` parses `3` as an integer literal type, and `count<N>` parses `N` as
a named type. The checker interprets each written argument by the **kind** of
the callee's parameter in that position:

- a const position accepts an integer literal, a module `const` name, or a
  const parameter name;
- a type position accepts a type.

A literal or const name at a type position, or a type at a const position, is
`E0813`. This keeps the grammar unchanged and avoids a second argument-list
production. It also means const arguments are not arbitrary expressions:
`Array<T, N + 1>`-style arguments are not part of this change.

A const name must resolve to a compile-time `int` when it is used as a const
argument. A local `let` is not a compile-time value and cannot be a const
argument; a module `const` and a const parameter can.

## Explicit arguments and inference

Version 1 has **no const argument inference**. A callee with a const parameter
must be written with its full generic argument list:

```xulo
count<3>()          // ok
count()             // error: a const argument is required
count<int>()        // error[E0813]: `int` is not a const argument
```

The existing all-or-nothing rule extends: a written list supplies every type
and const parameter of the callee, or none is written. A callee with a const
parameter cannot omit the list, because no argument supplies N. Type-only
generics keep the current inferred/explicit behaviour.

## Values and instantiation

- A const parameter is a compile-time value of type `int`; it may appear in
  ordinary expressions, constant expressions, and const arguments in the
  declaration's body. It has no runtime representation of its own.
- The as-if rule extends to const parameters: each distinct tuple of type and
  const arguments is a separate instantiation, with each type parameter
  replaced by its type and each const parameter by its integer value.
- Type identity uses the evaluated const value: `count<3>` and `count<M>`
  with `const M = 3` are the same instantiation; `count<3>` and `count<4>`
  differ.
- A const argument is evaluated at compile time; a module `const` used as an
  argument must be a compile-time `int`.

## Grammar and AST

`GenericParam` gains the const form:

```text
GenericParam = Identifier [ ":" TraitBound ]
             | "const" Identifier ":" Type ;
```

The `Type` after the colon is restricted to `int` by the checker (`E0812`).
`TypeArgs`/`TypeList` are unchanged; the checker performs the kind-directed
reading above. The AST records the parameter kind (type or const). A written argument keeps
its syntax node — an integer literal or a path — and the checker interprets
it by the callee's parameter kind; the AST itself need not decide.

## Diagnostics

| Code | Condition |
|------|-----------|
| `E0812` | a const parameter's declared type is not `int` |
| `E0813` | a generic argument has the wrong kind — a literal/const name at a type position, or a type at a const position |
| `E0807` | a written generic-argument list before a callee that declares no generic parameters (the existing row, extended from type parameters) |
| `E0808` | a written generic-argument list whose length differs from the callee's parameter count (the existing row, extended from type parameters) |

A const name that does not resolve, or a module const with a non-`int` type,
uses the ordinary unresolved-name/value-type diagnostics before the const
argument rule applies.

## Alternatives considered

### A special fixed-size type with literal-only lengths

**Cost.** `Array<int, 3>` as a built-in special case cannot be abstracted over:
`fn f<N>(a: Array<int, N>)` is impossible, so the feature is a family of
concrete types, not a generic language feature. Rejected as the mechanism;
a consumer type can follow after const generics.

### Type-level integer literal types

**Cost.** Reusing integer literal types as value-level lengths would make
`N` a type, not a value; it cannot be used in arithmetic in a body, and it
needs type-level equality/normalization semantics. A const parameter is the
direct expression of "a value in the type". Rejected for version 1.

### `usize` const parameters

**Cost.** The language's lengths and indices are `int`; `usize` has no
conversion to/from `int`. Keeping one numeric layer is simpler. Rejected.

### Const arithmetic in type arguments — `N + 1`

**Cost.** Const expression grammar, evaluation, and type normalization; a
large addition. Deferred. Ordinary arithmetic on N in a body is available.

### Const argument inference

**Cost.** Unification over const values and length evidence; deferred. A
caller writes the value explicitly in version 1.

### Bundle const generics with the first consumer

**Cost/benefit.** A consumer (fixed-length array) is the natural motivation,
but bundling a generic mechanism with a large type change makes review and
acceptance harder. This change is self-contained and testable through
const-parameter functions; the consumer follows as its own change.

## Consequences

### Grammar and AST

- `GenericParam` gains the const production; `TypeArgs` is unchanged. The AST
  and `machine-readable.md` record the parameter kind and const argument form.

### Typing and diagnostics

- `E0812` (bad const parameter type) and `E0813` (kind mismatch) are added;
  the existing `E0807`/`E0808` rows are extended to generic arguments.
- Type checking substitutes const values at instantiation; no runtime
  argument is introduced.
- Monomorphization can increase code size when a declaration is instantiated
  at many const values; no size promise is added.

### Documentation

- `types/generics.md` owns the const-parameter rules; `grammar.md` owns the
  production; `type-system/errors.md` owns the new codes; `ast.md` records the
  shape. A consumer type is a separate change.

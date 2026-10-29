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
| Argument | integer literal, a `const` binding (module or block), or a const parameter name; written explicitly |
| Use | N is a compile-time `int` value in the declaration's body; it is never a type |
| Application | a call to a declaration with const parameters requires the full generic argument list; an `impl` is matched from its implemented type |
| Arithmetic | not in type arguments in version 1; ordinary arithmetic on N in a body is allowed |
| Inference | none for const parameters |

A type parameter is still a type and never a value; a const parameter is the
converse. They share one parameter list and one namespace of names, and a
duplicate name is an error.

## Kind-directed arguments

The grammar keeps both existing argument-list productions:

```text
TypeArgs     = "<" TypeList ">" ;
CallTypeArgs = "<" [ TypeList ] ">" ;
TypeList     = Type { "," Type } ;
```

`TypeArgs` is the list of a named-type application (`Array<int, 3>`);
`CallTypeArgs` is the list of a call (`count<3>()`). Their elements are `Type`
nodes, so `3` parses as an integer literal type and `N` as a named type. The
checker interprets each written argument by the **kind** of the callee's
parameter in that position:

- a const position accepts an integer literal, a `const` binding, or a const
  parameter name, and evaluates it to a compile-time `int`;
- a type position accepts a type.

A const argument supplied for a type parameter, or a type argument supplied
for a const parameter, is `E0813`. A literal is not by itself a kind error:
the language's literal types remain usable as type arguments. An integer
literal, however, has no type meaning outside a const position; it is
admitted there only as a const argument. Names are resolved in the namespace
their position requires — the value namespace for a const argument and the
type namespace for a type argument — so a name that also exists in the other
namespace is not a kind mismatch.

This keeps the grammar unchanged and avoids a second argument-list
production. It also means const arguments are not arbitrary expressions:
`Array<T, N + 1>`-style arguments are not part of this change. Such material
does not parse as a type list; at a call site it falls back to comparison
syntax, and in a type annotation it is a syntax failure. A computed const
value is written through a `const` binding instead.

A negative value is written the same way: `-1` is a unary minus, not an
integer literal, so it is not a type-named const argument. `const M = -1`
followed by `Array<int, M>` is the form; the consumer's domain check then
reports a negative length.

A const name must resolve to a compile-time `int` when it is used as a const
argument. A local `let` is not a compile-time value and cannot be a const
argument; a `const` binding — at module level or in a block — and a const
parameter can.

## Explicit arguments and inference

Version 1 has **no const argument inference**. A callee with a const parameter
must be written with its full generic argument list:

```xulo
count<3>()          // ok
count()             // error[E0816]: a const argument is required
count<int>()        // error[E0813]: `int` is not a const argument
```

The existing all-or-nothing rule extends: a written list supplies every type
and const parameter of the callee, or none is written. A call to a callee
with a const parameter cannot omit the list, because no argument supplies N
(`E0816`). Type-only generics keep the current inferred/explicit behaviour.

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
- A const argument is evaluated at compile time; a `const` binding used as an
  argument must be a compile-time `int`.

### Const parameters on `impl`s

A generic `impl` MAY declare const parameters alongside type parameters. An
impl is never applied by writing an argument list: matching the implemented
type against the receiver's type instantiates its type and const parameters,
exactly as it already instantiates type parameters. The determined const
value is substituted in the impl's methods. This is impl matching, not
call-site const inference.

A const parameter on an `impl` MUST occur in the implemented type so that
matching determines it; an impl whose const parameter does not occur is
ill-formed (`E0817`). Two impls that can apply to the same type after
substitution conflict under the ordinary `E0805` rule.

### First-class uses

A function with const parameters cannot be used as a bare function value in
version 1: there is no value-position syntax for writing its const arguments,
and they are never inferred. Wrap it in a closure that supplies them, as in
`let f = fn(x: int): int { repeat<3>(x) }`; a bare use is `E0216`.

## Grammar and AST

`GenericParam` gains the const form:

```text
GenericParam = Identifier [ ":" TraitBound ]
             | "const" Identifier ":" Type ;
```

The `Type` after the colon is restricted to `int` by the checker (`E0812`).
`TypeArgs`, `CallTypeArgs`, and `TypeList` are unchanged; the checker
performs the kind-directed reading above. The AST records the parameter kind
(type or const). A written argument keeps its syntax node — an integer
literal or a path — and the checker interprets it by the callee's parameter
kind; the AST itself need not decide. A `TraitDeclaration`'s `genericParams`
remains always empty: a trait may not declare type or const parameters
(`E0810`).

## Diagnostics

| Code | Condition |
|------|-----------|
| `E0812` | a const parameter's declared type is not `int` |
| `E0813` | a generic argument has the wrong kind — a const argument supplied for a type parameter, or a type argument supplied for a const parameter |
| `E0807` | a written generic-argument list before a callee that declares no generic parameters (the existing row, extended from type parameters) |
| `E0808` | a written generic-argument list whose length differs from the callee's parameter count (the existing row, extended from type parameters) |
| `E0816` | a callee with const parameters is called without a written generic-argument list |
| `E0817` | a const parameter on an `impl` does not occur in the implemented type |

A const name that does not resolve, or a `const` binding with a non-`int`
type, uses the ordinary unresolved-name/value-type diagnostics before the
const argument rule applies. A trait that declares generic parameters is
`E0810`; an ill-formed type application is `E0230`; a type with generic
parameters that requests `#Derive` is `E0112`.

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

- `GenericParam` gains the const production; `TypeArgs`, `CallTypeArgs`, and
  `TypeList` are unchanged. The AST records the parameter kind and const
  argument form; `machine-readable.md` points at snapshots that must be
  regenerated from `grammar.md`.

### Typing and diagnostics

- `E0812` (bad const parameter type), `E0813` (kind mismatch), `E0816`
  (omitted const argument list), and `E0817` (undetermined impl const
  parameter) are added; `E0807`/`E0808` are extended to generic arguments;
  `E0810`, `E0112`, and `E0230` become generic-parameter rules.
- Type checking substitutes const values at instantiation; no runtime
  argument is introduced. Impl matching determines const values from the
  implemented type.
- Monomorphization can increase code size when a declaration is instantiated
  at many const values; no size promise is added.

### Documentation

- `types/generics.md` owns the const-parameter rules; `grammar.md` owns the
  production; `type-system/errors.md` owns the new codes; `ast.md` records the
  shape. `functions.md`, `names.md`, `types/traits.md`, `types/enums.md`,
  `types/README.md`, `types/function-types.md`, `expressions/calls.md`,
  `builtins/README.md`, and the `README.md` Terminology table receive the
  generic-parameter wording and the const-parameter/value entries they need. A
  consumer type is a separate change.

# Add Const Generics

**Status:** draft · **Date:** 2026-10-27 · **Affects:**
`grammar.md`, `types/generics.md`, `type-system/checking-rules.md`,
`type-system/errors.md`, `ast.md`

## Summary

Xulo's generic parameters stand for types. This change adds **const
parameters** of type `int`, so a declaration can be parameterized by a
compile-time integer value as well as by types:

```xulo
fn count<const N: int>(): int { N }
fn repeat<const N: int>(value: int): int { N * value }
let three = count<3>()               // 3
```

The parameter list mixes type and const parameters — `<T, const N: int>` — and
each application writes the full argument list. Const arguments are integer
literals, module `const` names, or const parameter names, written explicitly;
version 1 has no const arithmetic in argument position and no const argument
inference. A const parameter is a compile-time value usable in the
declaration's body and substituted at each instantiation; it is never a type,
and a type parameter is never a value.

The first intended consumer is a fixed-length array in a later change; this
change defines only the generic mechanism and its diagnostics.

## Motivation

### Type parameters cannot express a length

A fixed-size type needs the size in the type: `Array<T, 3>` and `Array<T, 4>`
must be different types, and generic code must abstract over N. Xulo's
`<T>` parameters are type names only; an integer literal or const name in a
type-argument position is not currently a value parameter. Without const
generics, a fixed array can only be a family of special types or a tuple, and
generic functions over length are impossible.

### A value parameter is distinct from a type parameter

The language already distinguishes types from values: a type parameter may
appear in a type position and never in an expression; a value may appear in an
expression. A const parameter makes that distinction explicit in the generic
list: `N` is a value of type `int` and may appear wherever an integer
expression or a const argument may appear; it may not be applied to type
arguments or used as a type. Keeping the two kinds separate avoids the
ambiguity of "a parameter that is sometimes a type and sometimes a value."

### Explicit, small, and predictable

Version 1 accepts only integer literals, module `const` names, and const
parameter names as const arguments, and requires the full generic argument
list at every application. This avoids type-level arithmetic, const
normalization, and inference over values while still giving generic code a
length/parameter it can pass and use. Later changes can add `N + 1` and
inference without changing the declaration syntax.

## Scope

**In scope.** Const parameters `<const N: int>` on functions, `struct`s,
`enum`s, type aliases, and `impl` blocks; mixing them with type parameters;
const arguments written as integer literals, module `const` names, or const
parameter names; explicit all-or-nothing generic argument lists; const
parameters as compile-time values in bodies; type identity and
monomorphization over const values; the new diagnostics.

**Out of scope.** A fixed-length array type or any other consumer (separate
changes); const arithmetic in type arguments (`N + 1`); const argument
inference; const parameters of types other than `int`; const parameters on
`trait`s; default const arguments; type-level literal types.

## Compatibility

- No existing program changes meaning: the syntax is new only in a generic
  parameter list, and an application that writes a literal or const name where
  a const parameter is expected did not type-check before.
- A declaration with const parameters must write the full generic argument
  list at every application; a call that omits it is an error, not an
  inference. Existing type-parameter-only declarations keep their inference
  and optional explicit list.
- Const parameters are values; they do not reserve module-scope names and do
  not affect the type/value namespaces.
- The change is independent of any module or standard-library API.

# Prelude

The prelude is what a module knows before it imports anything: the built-in
types, the built-in protocol, and the built-in namespaces. Nothing declares
the prelude, no import introduces it, and no import can take it away; it is
the constant background of name resolution
([`../names.md`](../names.md)). This chapter lists what the prelude contains
and the names it reserves. The built-in types' members are specified with the
types; the compiler-provided functions and the namespace members are specified
in [`intrinsic-functions.md`](intrinsic-functions.md).

## What the prelude contains

### Built-in types

Every name in this table is in scope in every module, wherever a type may be
written; `null` additionally has a literal form usable in expression position.

| Type | Description | Chapter |
|------|-------------|---------|
| `bool` | the two truth values, `true` and `false` | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `string` | immutable UTF-8 sequence of Unicode scalar values | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `int` | 64-bit signed integer; the type of integer literals | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `float` | 64-bit IEEE-754 binary64 floating point | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `number` | the user-facing numeric type, the union `int \| float` | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64` | fixed-bit numeric types for exact layout and foreign interfaces | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `unknown` | the top type: a value of any type, whose type is not known | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `List<T>` | ordered, growable sequence of elements of one type | [`../types/composite-types.md`](../types/composite-types.md) |
| `Array<T, N>` | inline fixed-length sequence of N elements; N is a const parameter, N ≥ 0 | [`../types/composite-types.md`](../types/composite-types.md) |
| `Map<K, V>` | insertion-ordered dictionary with unique keys | [`../types/composite-types.md`](../types/composite-types.md) |
| `Set<T>` | unordered collection of unique values | [`../types/composite-types.md`](../types/composite-types.md) |
| `unit` | the type of expressions that produce no meaningful value | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `null` | the sole value of the type `null`; a member of every `T?` | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `Task<T>` | the value an `async` call produces; unwrapped by `await` | [`../types/function-types.md`](../types/function-types.md) |
| `Range<T>` | the value produced by `..<` and `...`; iterable by `for` | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `Result<T, E>` | the built-in result of a fallible operation: `Ok(T)` or `Err(E)` | [`../error-handling.md`](../error-handling.md) |
| `View` | the marker type a component returns | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `Error` | the conventional error payload of `Result<T, Error>` | [`../error-handling.md`](../error-handling.md) |

`struct`, `enum`, `trait`, and `type` declarations introduce further types, but
a program declares them itself; they are not part of the prelude (see
[`../types/README.md`](../types/README.md)).

### Built-in protocols

| Protocol | Method | Required by |
|----------|--------|-------------|
| `ToString` | `fn toString(self): string` | interpolation in a template literal, and the intrinsic `str` |

- A type implements `ToString` with `impl ToString for T`, whose `toString`
  returns the string form of the receiver
  ([`../types/traits.md`](../types/traits.md)).
- The base types — `int`, `float`, `bool`, `string` — convert to text
  without an implementation; every other type needs one.
- A value interpolated by `${…}` MUST be a base type or implement
  `ToString` ([`../expressions/literals.md`](../expressions/literals.md));
  the same requirement is stated for `str` in
  [`intrinsic-functions.md`](intrinsic-functions.md#conversion).

### Built-in namespaces

| Namespace | Contents | Specified in |
|-----------|----------|--------------|
| `Math` | mathematical constants and functions | [`Math` namespace](intrinsic-functions.md#math-namespace) |
| `Time` | timestamps and async sleep | [`Time` namespace](intrinsic-functions.md#time-namespace) |
| `Task` | task combinators `all`, `race`, `resolve`, `timeout` | [`Task` namespace](intrinsic-functions.md#task-namespace) |

Members are selected with `.`, exactly like fields and methods:
`Math::PI`, `Time::now()`, `Task::all(tasks)`
([`../expressions/path-and-access.md`](../expressions/path-and-access.md)).

### Reserved names

A module-scope declaration — `fn`, `struct`, `enum`, `trait`, `type`, `let`,
`let mut`, or `const` — MUST NOT bear any of the names below. Declaring one is
a compile-time error (`E0106`) reported under the diagnostic category for reserved names
in [`../type-system/errors.md`](../type-system/errors.md).

| Category | Names |
|----------|-------|
| Value and marker types | `List`, `Array`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error` |
| Protocols | `ToString` |
| Namespaces | `Math`, `Time`, `Task` |
| Intrinsic functions | `print`, `println`, `str`, `assert` |

`Task` appears twice because it is both a type and a namespace. Every
intrinsic listed in
[`intrinsic-functions.md`](intrinsic-functions.md#complete-signature-index)
is reserved under the same rule.

## Built-in members

The built-in collections (`List`, `Array`, `Map`, `Set`) and `string` expose
inherent members, specified with their types: `List`, `Array`, `Map`, and
`Set` in [`../types/composite-types.md`](../types/composite-types.md), and
`string` in [`../types/primitive-types.md`](../types/primitive-types.md). They
are not prelude operations: they have no unqualified spelling, no import
introduces them, and they are not module-scope names. A local binding named
`len` or `get` does not shadow `xs.len` or `m.get`; a static member is reached
through a type name, as in `Map::fromEntries`.

## Prelude visibility

- The prelude is in scope in every module from its first line; no import is
  required, and no form of `import` introduces it, disables it, or removes any
  part of it.
- Prelude names are consulted only after every earlier scope of the resolution
  order has failed to provide the name
  ([`../names.md`](../names.md); see [README](README.md#resolution-order)).
- A prelude name may become unreachable in a region of a file only through
  shadowing, and only for the duration of the shadowing scope.

## Shadowing

- **Module scope.** A module-scope declaration MUST NOT bear a reserved name
  ([Reserved names](#reserved-names)); redeclaring one is a compile-time error
  (`E0106`), not a shadowing.
- **Inner scopes.** An inner-scope `let`, `let mut`, `const`, or parameter MAY
  shadow an intrinsic name. From its declaration to the end of the enclosing
  scope the name denotes that binding alone, so a call written there reaches
  the local value rather than the intrinsic; outside the scope the intrinsic
  name is visible again. Built-in members are reached only after a receiver,
  so no local binding hides `xs.len` or `m.get`.

  ```xulo
  fn demo(): int {
    let str = "not the intrinsic"    // hides `str` for the rest of this block
    print(str)
    42
  }
  ```

- **Type names.** A binding whose spelling is a built-in type name hides that
  type from its declaration to the end of the enclosing scope. Within that
  scope the name in a type position — an annotation, a type argument, a bound —
  resolves to the binding, which does not denote a type, so the use is a
  compile-time error (`E0101`) ([`../names.md`](../names.md)). Shadowing therefore never
  redefines a built-in type and never changes what a type position denotes: it
  only makes the built-in type unavailable there, which is why a scope that
  needs `List<T>` in type position MUST NOT shadow `List`.
- Type parameters and field names are ordinary identifiers, and a type
  parameter named `Set` or `Map` follows the rule above
  ([`../types/primitive-types.md`](../types/primitive-types.md)).

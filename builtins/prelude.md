# Prelude

The prelude is what a module knows before it imports anything: the built-in
types, the built-in protocol, and the operations defined directly on the
built-in collections. Nothing declares the prelude, no import introduces it,
and no import can take it away; it is the constant background of name
resolution ([`../names.md`](../names.md)). This chapter lists what the
prelude contains and specifies the prelude operations. The compiler-provided
functions and the namespace members among them are specified in
[`intrinsic-functions.md`](intrinsic-functions.md).

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
| Value and marker types | `List`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error` |
| Protocols | `ToString` |
| Namespaces | `Math`, `Time`, `Task` |
| Intrinsic functions | `print`, `println`, `str`, `assert` |

`Task` appears twice because it is both a type and a namespace. The prelude
operations defined in the next section are reserved under the same rule, and
so is every intrinsic listed in
[`intrinsic-functions.md`](intrinsic-functions.md#complete-signature-index).

## Prelude operations

Iteration, subscripts, and concatenation on the collections are core syntax and
are specified with the types themselves. Everything else that the built-in
collections need — removal, size, entry tests, lookup, append, enumeration,
bulk conversion, and the `Set` constructor — is a prelude operation: an
unqualified function in scope in every module, called like any other function.

```xulo
let mut counts: Map<string, int> = { "a": 1, "b": 2 }
print(mapSize(counts))              // 2
print(mapHasKey(counts, "a"))      // true
print(mapGet(counts, "z"))          // null: absent, and no failure
let previous = mapRemove(counts, "a")   // counts loses "a"; previous is 1

let tags: Set<string> = setFromList(["a", "b", "a"])
print(setSize(tags))                // 2: the duplicate collapsed
let added = setInsert(tags, "c")
print(setContains(tags, "c"))       // true

let mut xs = ["a", "b"]
listPush(xs, "c")                   // xs is ["a", "b", "c"]
let removed = listRemove(xs, 0)     // "a"; xs is ["b", "c"]
for e in enumerate(xs) {
  print(`${e.0}=${e.1}`)             // 0=b, then 1=c
}

let user = { name: "lyy", age: 30 }  // a Map<string, string | int>
print(user["name"])                  // lyy
```

### Map operations

| Signature | Meaning |
|-----------|---------|
| `mapSize<K, V>(m: Map<K, V>): int` | the number of entries in `m` |
| `mapHasKey<K, V>(m: Map<K, V>, k: K): bool` | `true` when `k` has a binding in `m` |
| `mapGet<K, V>(m: Map<K, V>, k: K): V?` | the value `k` is bound to, or `null` when `k` has no binding |
| `mapRemove<K, V>(m: mut Map<K, V>, k: K): V?` | removes the binding for `k` and returns the value it had, or `null` when `k` was absent |
| `mapToEntries<K, V>(m: Map<K, V>): List<(K, V)>` | one `(key, value)` tuple per entry, in insertion order (bulk conversion) |
| `mapFromEntries<K, V>(entries: List<(K, V)>): Map<K, V>` | builds a map from `(key, value)` tuples, in list order; a repeated key keeps the later entry |

- Reading (`m[k]`), writing (`m[k] = v`), and iterating the keys (`for k in m`)
  are core syntax and need no prelude function
  ([`../types/composite-types.md`](../types/composite-types.md)).
- `mapGet` is the lookup that cannot fail: reading `m[k]` for a key with no
  binding is a runtime failure, while `mapGet(m, k)` answers `null`
  ([`../error-handling.md`](../error-handling.md)).
- `mapRemove` mutates its first argument, so that argument MUST be a mutable
  place — a `let mut` binding or a `mut` parameter; passing anything else is a
  compile-time error (`E0401`).

### Set operations

The language defines no set literal, so the prelude provides the constructor
and the operations that manipulate a set.

| Signature | Meaning |
|-----------|---------|
| `setFromList<T>(values: List<T>): Set<T>` | constructs a set holding each distinct element of `values` exactly once |
| `setInsert<T>(s: mut Set<T>, v: T): bool` | adds `v` to `s`; returns `true` when `v` was not already present |
| `setRemove<T>(s: mut Set<T>, v: T): bool` | removes `v` from `s`; returns `true` when `v` was present |
| `setContains<T>(s: Set<T>, v: T): bool` | `true` when `v` is a member of `s` |
| `setSize<T>(s: Set<T>): int` | the number of elements in `s` |

- `setInsert` and `setRemove` mutate their first argument, which MUST
  therefore be a mutable place, exactly as for `mapRemove`.
- A set has no specified order: neither `setFromList` nor any other
  operation defines the order of the elements, and programs MUST NOT depend on
  one ([`../types/composite-types.md`](../types/composite-types.md)).

### List operations

`+`, the subscript, and iteration on a list are core syntax
([`../types/composite-types.md`](../types/composite-types.md)); the prelude
supplies the operations they do not cover:

| Signature | Meaning |
|-----------|---------|
| `listSize<T>(xs: List<T>): int` | the number of elements in `xs` |
| `listPush<T>(xs: mut List<T>, v: T): unit` | appends `v` to the end of `xs` |
| `listRemove<T>(xs: mut List<T>, i: int): T?` | removes the element at index `i`, shifting later elements down, and returns it; `null` when `i` is negative or at or past the end |
| `enumerate<T>(xs: List<T>): List<(int, T)>` | one `(index, element)` tuple per element, in order |

- `listPush` and `listRemove` mutate their first argument, which MUST
  therefore be a mutable place, exactly as for `mapRemove`.
- `listRemove` is the removal that cannot fail: the subscript `xs[i]` for an
  out-of-range index is a runtime failure, while `listRemove(xs, i)` answers
  `null` and leaves `xs` unchanged.

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
  shadow a prelude operation or an intrinsic name. From its declaration to the
  end of the enclosing scope the name denotes that binding alone, so a call
  written there reaches the local value rather than the prelude; outside the
  scope the prelude name is visible again.

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

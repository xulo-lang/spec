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
| `boolean` | the two truth values, `true` and `false` | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `string` | immutable UTF-8 sequence of Unicode scalar values | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `int` | 64-bit signed integer; the type of integer literals | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `float` | 64-bit IEEE-754 binary64 floating point | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `number` | the user-facing numeric type, the union `int \| float` | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64` | fixed-bit numeric types for exact layout and foreign interfaces | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `unknown` | the top type: a value of any type, whose type is not known | [`../types/primitive-types.md`](../types/primitive-types.md) |
| `list<T>` | ordered, growable sequence of elements of one type | [`../types/composite-types.md`](../types/composite-types.md) |
| `map<K, V>` | insertion-ordered dictionary with unique keys | [`../types/composite-types.md`](../types/composite-types.md) |
| `set<T>` | unordered collection of unique values | [`../types/composite-types.md`](../types/composite-types.md) |
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
| `ToString` | `fn to_string(self): string` | interpolation in a template literal, and the intrinsic `str` |

- A type implements `ToString` with `impl ToString for T`, whose `to_string`
  returns the string form of the receiver
  ([`../types/traits.md`](../types/traits.md)).
- The base types — `int`, `float`, `boolean`, `string` — convert to text
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
| `Task` | task combinators `all`, `race`, `resolve` | [`Task` namespace](intrinsic-functions.md#task-namespace) |

Members are selected with `.`, exactly like fields and methods:
`Math.PI`, `Time.now()`, `Task.all(tasks)`
([`../expressions/path-and-access.md`](../expressions/path-and-access.md)).

### Reserved names

A module-scope declaration — `fn`, `struct`, `enum`, `trait`, `type`, `let`,
`let mut`, or `const` — MUST NOT bear any of the names below. Declaring one is
a compile-time error reported under the diagnostic category for reserved names
in [`../type-system/errors.md`](../type-system/errors.md).

| Category | Names |
|----------|-------|
| Base and numeric types | `boolean`, `string`, `int`, `float`, `number`, `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64` |
| Value and marker types | `null`, `unit`, `unknown`, `list`, `map`, `set`, `Task`, `Range`, `Result`, `View`, `Error` |
| Protocols | `ToString` |
| Namespaces | `Math`, `Time`, `Task` |
| Intrinsic functions | `print`, `println`, `str` |

`Task` appears twice because it is both a type and a namespace. The prelude
operations defined in the next section are reserved under the same rule, and
so is every intrinsic listed in
[`intrinsic-functions.md`](intrinsic-functions.md#complete-signature-index).

## Prelude operations

Iteration, subscripts, and concatenation on the collections are core syntax and
are specified with the types themselves. Everything else that the built-in
collections need — removal, size, entry tests, bulk conversion, and the `set`
constructor — is a prelude operation: an unqualified function in scope in every
module, called like any other function.

```xulo
let mut counts: map<string, int> = { "a": 1, "b": 2 }
print(map_size(counts))              // 2
print(map_has_key(counts, "a"))      // true
let previous = map_remove(counts, "a")   // counts loses "a"; previous is 1

let tags: set<string> = set_from_list(["a", "b", "a"])
print(set_size(tags))                // 2: the duplicate collapsed
let added = set_insert(tags, "c")
print(set_contains(tags, "c"))       // true

let user = { name: "lyy", age: 30 }  // a map<string, string | int>
print(user["name"])                  // lyy
```

### Map operations

| Signature | Meaning |
|-----------|---------|
| `map_size<K, V>(m: map<K, V>): int` | the number of entries in `m` |
| `map_has_key<K, V>(m: map<K, V>, k: K): boolean` | `true` when `k` has a binding in `m` |
| `map_remove<K, V>(m: mut map<K, V>, k: K): V?` | removes the binding for `k` and returns the value it had, or `null` when `k` was absent |
| `map_to_entries<K, V>(m: map<K, V>): list<(K, V)>` | one `(key, value)` tuple per entry, in insertion order (bulk conversion) |
| `map_from_entries<K, V>(entries: list<(K, V)>): map<K, V>` | builds a map from `(key, value)` tuples, in list order; a repeated key keeps the later entry |

- Reading (`m[k]`), writing (`m[k] = v`), and iterating the keys (`for k in m`)
  are core syntax and need no prelude function
  ([`../types/composite-types.md`](../types/composite-types.md)).
- `map_remove` mutates its first argument, so that argument MUST be a mutable
  place — a `let mut` binding or a `mut` parameter; passing anything else is a
  compile-time error.

### Set operations

The language defines no set literal, so the prelude provides the constructor
and the operations that manipulate a set.

| Signature | Meaning |
|-----------|---------|
| `set_from_list<T>(values: list<T>): set<T>` | constructs a set holding each distinct element of `values` exactly once |
| `set_insert<T>(s: mut set<T>, v: T): boolean` | adds `v` to `s`; returns `true` when `v` was not already present |
| `set_remove<T>(s: mut set<T>, v: T): boolean` | removes `v` from `s`; returns `true` when `v` was present |
| `set_contains<T>(s: set<T>, v: T): boolean` | `true` when `v` is a member of `s` |
| `set_size<T>(s: set<T>): int` | the number of elements in `s` |

- `set_insert` and `set_remove` mutate their first argument, which MUST
  therefore be a mutable place, exactly as for `map_remove`.
- A set has no specified order: neither `set_from_list` nor any other
  operation defines the order of the elements, and programs MUST NOT depend on
  one ([`../types/composite-types.md`](../types/composite-types.md)).

`list<T>` needs no prelude operations of its own: `+`, the subscript, and
iteration are core syntax
([`../types/composite-types.md`](../types/composite-types.md)).

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
  ([Reserved names](#reserved-names)); redeclaring one is a compile-time error,
  not a shadowing.
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
  compile-time error ([`../names.md`](../names.md)). Shadowing therefore never
  redefines a built-in type and never changes what a type position denotes: it
  only makes the built-in type unavailable there, which is why a scope that
  needs `list<T>` in type position MUST NOT shadow `list`.
- Type parameters and field names are ordinary identifiers, and a type
  parameter named `set` or `map` follows the rule above
  ([`../types/primitive-types.md`](../types/primitive-types.md)).

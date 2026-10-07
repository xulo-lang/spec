# Enums

An `enum` declaration introduces a new *nominal* type together with a fixed set
of *variants*, each of which may carry associated values. Enum values are
created through variant paths, deconstructed with `match`, and compared
structurally. This chapter covers declaration, construction, matching, and the
ways enums relate to unions and to the error model of
[`../error-handling.md`](../error-handling.md).

## Declaration

```xulo
enum Theme {
  Light,
  Dark,
  System,
}
```

Rules:

- An `enum` declaration is `enum` Identifier `{` variants `}`. A `,` MAY be
  written between variants (and a trailing `,` is permitted before the closing
  `}`); because variant names cannot start with `,` or `}`, omitting the comma
  is unambiguous and equally valid. An
  `enum` MAY declare no variants, in which case no value of the type exists.
- Enums are nominal: two enums that list the same variants are still distinct
  types, and a value of one is never usable where the other is expected. The
  enum name is a type name and follows the naming conventions of
  [`../names.md`](../names.md), as do the variant names.
- An enum is generic when a type parameter list follows its name — for example
  `enum Result<T, E> { Ok(T), Err(E) }` — and the parameters are then in scope in
  every payload type.
- Visibility uses `pub` (see [`../modules/README.md`](../modules/README.md)):
  `pub enum …` exports the enum and its variants from the module, and without
  `pub` the enum is private to it. Variants have no visibility modifier of their
  own; they are exported exactly when their enum is.

## Variants without payload

A variant that carries no data is a namespaced constant of its enum type.

```xulo
let theme = Theme::Dark
```

Rules:

- `::` is the variant path separator, in expressions and in patterns alike (see
  [`../expressions/path-and-access.md`](../expressions/path-and-access.md)).
  `Theme::Dark` is a path that denotes a constant value whose type is `Theme`.
- Variants are namespaced under their enum: they are **not** top-level names.
  `Dark` alone is not in scope anywhere, and the spelling `Theme.Dark` is
  ill-formed — `.` is reserved for field access, method calls, and namespace
  members.
- A variant path does not introduce a type. There is no type `Dark`, and no
  expression has the type "the variant `Dark`"; the type of the expression is
  the enum itself.
- Every payload-less variant of an enum has the same type — the enum itself —
  so only `match` and `==` distinguish them.

## Variants with associated values

A variant may declare associated values, written as a comma-separated parameter
list after the variant name. Two parameter forms exist.

```xulo
enum Shape {
  Circle(float),
  Rect(width: float, height: float),
}

enum Operation {
  Success(data: number),
  Error(message: string),
}

enum Person {
  Nobody,
  Named(string, int),
}
```

Rules:

- The *positional* form writes only a type per slot: `Circle(float)`,
  `Named(string, int)`. The *named* form labels each slot:
  `Success(data: number)`, `Rect(width: float, height: float)`.
- Within a single variant, all parameters MUST use the same form; a variant may
  not mix labeled and unlabeled parameters. Across variants, an enum MAY mix
  payload-less variants, positional payloads, and named payloads freely, as
  `Shape` and `Person` do.
- The names of a named payload label the slots: they document the payload and
  appear in diagnostics, but they are not argument labels, because construction
  and matching are always positional. A payload slot may have any type —
  `struct`s, `enum`s, `list`s, `object`s, and type parameters alike.
- Construction applies arguments positionally, exactly as many as the variant
  declares: `Shape::Circle(2.0)`, `Shape::Rect(3.0, 4.0)`,
  `Person::Named("Ada", 36)`. The arity MUST match the declaration, and each
  argument MUST be assignable to its declared slot type.
- The resulting value has the enum type — `Shape::Circle(2.0)` has type
  `Shape`, not a variant-specific type. There is no first-class type for an
  individual variant.
- A variant that declares a payload MUST be applied to arguments. Referring to
  it without a parameter list, as in `Shape::Circle`, is ill-formed: a variant
  path is a value constructor, not a function.

## Pattern matching

`match` deconstructs an enum value, binds its payloads, and decides which case a
value belongs to.

```xulo
fn area(s: Shape): float {
  match s {
    Shape::Circle(r) => 3.14159 * r * r,
    Shape::Rect(w, h) => w * h,
  }
}
```

Rules:

- A deconstruction pattern is `Enum::Variant(pattern, …)`. Each slot pattern is
  either a binding name, which introduces an immutable binding scoped to the arm
  with the declared type of that slot, or `_`, which discards it. Patterns
  deconstruct recursively — a slot may itself be a variant path with further
  slots — so nested values are matched in one arm:

  ```xulo
  let p = Person::Named("Ada", 36)
  let name = match p {
    Person::Named(n, _) => n,
    Person::Nobody => "anonymous",
  }
  ```

- When the scrutinee is a number, an arm MAY use a literal or a range pattern;
  range patterns use the range operators of the expression language with their
  usual inclusivity, `0...29` (closed) and `30..<60` (half-open):

  ```xulo
  let band = match score {
    0...29 => "low",
    30...69 => "medium",
    _ => "high",
  }
  ```

- A `match` whose scrutinee has an enum type MUST be exhaustive: every variant
  must be covered, either by an arm naming it or by a catch-all `_` arm. A
  `match` that leaves a variant uncovered is a compile-time error; see
  [`../type-system/errors.md`](../type-system/errors.md).

  ```xulo
  let label = match theme {
    Theme::Dark => "dark",
    _ => "other",
  }
  ```

- The result types of all arms are checked together; they MUST be mutually
  compatible when the `match` is used as an expression (see
  [`../type-system/checking-rules.md`](../type-system/checking-rules.md)).
  The patterns, arm syntax, exhaustiveness, and use of `match` as an expression
  are specified in
  [`../expressions/control-flow.md`](../expressions/control-flow.md).

An `enum` is the usual way to declare a closed set of failure modes, because
exhaustiveness then guarantees that every failure is handled:

```xulo
enum LoadError { NotFound(path: string), Denied(user: string), Timeout(ms: int) }

fn message(e: LoadError): string {
  match e {
    LoadError::NotFound(p) => "not found: " + p,
    LoadError::Denied(u) => "denied: " + u,
    LoadError::Timeout(ms) => "timed out after " + str(ms) + " ms",
  }
}
```

## Conversion and testing

- `match` is the sole *variant test*: it is the only construct that selects a
  case by variant, independently of the payloads. The language defines no `is`
  operator, no conversion operator between enum types, and no implicit
  conversion from an enum to any other type.
- Outside `match`, the only comparison available is `==`/`!=` against a value of
  the same enum type. It compares the whole value — variant and payloads — so for
  a payload-less variant it coincides with a variant test
  (`if theme == Theme::Dark { … }`).
- To test only the variant of a payload-carrying value outside a `match`
  statement, use a `match` expression that yields a `boolean`:

  ```xulo
  let succeeded = match op {
    Operation::Success(_) => true,
    _ => false,
  }
  ```

- An enum value is never implicitly converted to `string`. Interpolation of an
  enum value requires an `impl ToString for …` as described in
  [`traits.md`](traits.md).

## Comparison and equality

- `==` and `!=` accept two operands of the same enum type (after alias
  expansion) and compare them structurally: first the variant, then the
  payloads in slot order.
- Payloads are compared with `==` themselves, so an enum is comparable exactly
  when its slot types are: `list<T>`, `map<K, V>`, and `set<T>` payloads
  compare element-wise when the element, key, and value types do, and optional
  slots compare `null` against `null`.
- Operands of two different enum types MUST NOT be compared, even if their
  variant lists coincide, and relational operators (`<`, `>`, `<=`, `>=`) are not
  defined for enum values; both are type errors.

## Enums as error types

An error enum states a domain's failure modes as data, so that handling them is
exhaustive and checked. The two idioms are a dedicated error enum, matched
directly as `LoadError` above, and a result enum pairing a value with a possible
failure:

```xulo
enum Result<T, E> { Ok(T), Err(E) }

fn half(n: int): Result<int, string> {
  if n % 2 == 0 { Result::Ok(n / 2) } else { Result::Err("odd input") }
}
```

Thrown values, `try`/`catch`, and the built-in base error type `Error` are
specified in [`../error-handling.md`](../error-handling.md); domain errors are
conventionally declared as `enum`s so that both `match` and `catch` clauses can
name them precisely.

## Enums vs union types

Enums and the union type `T | U` overlap in purpose but differ in kind.

| | `enum` | `T \| U` |
|---|--------|----------|
| Naming | introduces a new nominal type and new case names | names no new type; combines existing types |
| Membership | decided by which variant was constructed | decided by the type of the value |
| Data | each case may declare its own payload, slot by slot | each case is a whole existing type |
| Checking | exhaustive `match` over a fixed set of cases | narrowing works over whatever types the union lists |
| Typing | nominal — two identical declarations are unrelated | structural — aliases are transparent |

Use an `enum` when introducing a domain concept with a closed set of cases —
states, commands, results, errors. Use a union to combine types that already
exist, such as `string | int`, or an optional `T?`, the shorthand for
`T | null`. A union invents no case names and carries no per-case payloads, and
an enum cannot be widened to unrelated types; the relations between unions,
intersections, and their members are defined in
[`type-relations.md`](type-relations.md).

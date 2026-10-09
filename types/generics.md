# Generics

A *generic* declaration is parameterized by *type parameters*: names that stand
for types chosen at each use of the declaration. This chapter defines where type
parameters may be declared, how they are bounded, how their type arguments are
determined at a call site, and what a generic definition means once those
arguments are known. The built-in parameterized types are described in
[`primitive-types.md`](primitive-types.md) and [`composite-types.md`](composite-types.md);
the relations between the resulting types are in [`type-relations.md`](type-relations.md).

## Type Parameters

A type parameter list is written `<` … `>` immediately after the name of the
declaration. Functions, `struct`s, `enum`s, `type` aliases, and `impl` blocks
MAY all be parameterized.

```xulo
fn first<T>(xs: list<T>): T { xs[0] }
struct Pair<K, V> { key: K, value: V }
enum Either<L, R> { Left(L), Right(R) }
type ApiResponse<T> = { data: T?, error: string? }

struct Wrapper<T> { value: T }
impl<T> Wrapper<T> {
  fn get(self): T { self.value }
}
```

Rules:

- A declaration has **at most one** type parameter list, written directly after
  the declaration name: `<T>`, `<K, V>`, `<T, E>`. It is comma-separated and its
  names MUST be distinct — `<T, T>` is ill-formed. Each name is in scope for the
  remainder of the declaration: the signature, the body, and — for `struct`,
  `enum`, `type` — every field, payload, and aliased type.
- A type parameter is a type name: it MAY be used wherever a type is expected
  (parameter types, field types, return types, bounds, nested generic
  arguments). It is never a value: a type parameter has no runtime
  representation and never occurs in an expression position.
- Parameter names follow the conventions of [`../names.md`](../names.md): a
  single uppercase letter (`T`, `U`, `K`, `V`) for a simple parameter, or an
  UpperCamelCase descriptive name. These are conventions, not grammar rules.

### Nested angle brackets

`>>` is a single token (the right-shift operator), so two closing angle
brackets MUST NOT be written adjacently. Whenever two `>` characters would
touch, a space MUST be written between them:

```xulo
let nested: list<map<string, list<int> > > = []
```

| Form | Well-formed |
|------|-------------|
| `list<list<T> >` | yes |
| `list<list<T>>` | no — `>>` would be read as the shift token |
| `list<list<list<T> > >` | yes |

The rule applies at every generic argument position, including in bounds.

## Bounds

A *bound* restricts the set of type arguments a parameter accepts. Bounds are
written either inline after the parameter name or in a `where` clause that
follows the whole signature.

```xulo
fn area_of<T: Area>(shape: T): float { Area.area(shape) }
fn perimeter_of<T>(shape: T): float where T: Area { Area.perimeter(shape) }
```

Rules:

- The forms are equivalent: `<T: Area>` and `where T: Area` constrain the same
  parameter in the same way, and a parameter MUST NOT be bounded in both places.
  A `where` clause follows the parameter list, parameter types, and declared
  return type, and precedes the body; it is a comma-separated list of bounds, one
  per parameter: `fn f<A, B>(x: A, y: B) where A: Area, B: Scalable { … }`.
- Multiple bounds on one parameter are joined with `+`: `<T: Area + Scalable>`,
  equivalently `where T: Area + Scalable`. The parameter is admissible only if
  its type argument satisfies **every** bound.
- A bound MUST name a `trait` visible at the declaration site; naming anything
  else is a compile-time error (see [`../type-system/errors.md`](../type-system/errors.md)).
- A bound is a *constraint*, not a subtyping relation: `T` with bound `Area` does
  not thereby become a subtype of `Area`. Inside the body, values of type `T`
  expose the members declared by `Area`, and explicit dispatch through the bound
  is well-formed ([`traits.md`](traits.md)).
- A bound is never written with `&`: `T & U` is an intersection *type*, which
  combines types a value already has, whereas `T: A + B` restricts which types
  may be substituted for a parameter.

**Bound checking at call sites.** Every type argument determined for a call MUST
satisfy every bound of the corresponding parameter. A call that would instantiate
a parameter with a type that does not implement one of its bounds is ill-formed
and MUST be reported with a diagnostic, even when the type argument was inferred
rather than written; see [`../type-system/checking-rules.md`](../type-system/checking-rules.md).

## Inference

Generic type arguments are never written at a call site; they are always
*inferred* from the types of the actual arguments.

```xulo
let n = first([1, 2, 3])    // T = int
let s = first(["a", "b"])   // T = string
```

Rules:

- The language has **no syntax for explicit type arguments in a call**:
  `first<int>(…)` and `map<string, int>(…)` are ill-formed — the call production
  of the grammar has no type argument list ([`../grammar.md`](../grammar.md)).
- Type arguments are determined by *unification* between the declared parameter
  types and the argument types. Unification decomposes type constructors
  structurally:

  | Constraint | Result |
  |------------|--------|
  | `list<T>` ≟ `list<int>` | `T = int` |
  | `map<K, V>` ≟ `map<string, int>` | `K = string`, `V = int` |
  | `T` ≟ `fn(int): boolean` | `T = fn(int): boolean` |
  | `T?` ≟ `string?` | `T = string` |
  | `list<T>` ≟ `map<string, int>` | fails (different heads) |

- A parameter occurring twice in a signature receives one solution; if two
  constraints demand different types for it, inference fails with a diagnostic.
- An *expected type* MAY determine what the arguments leave undetermined — an
  empty list literal, a numeric literal, a closure with unannotated parameters.
  It MUST NOT override a type already fixed by the arguments: in
  `let bad: string = first([1, 2])`, `T` is `int`, so the declaration is an
  error rather than an invitation to choose `T = string`.
- Inference MUST fail with a diagnostic when a parameter remains
  under-constrained:

  ```xulo
  fn empty<T>(): list<T> { [] }
  let xs = empty()             // error: T is not determined
  let ys: list<int> = empty()  // OK: T = int from the expected type
  ```

  Since explicit type arguments do not exist, such a call is repaired only by
  annotating its context. Constraint generation and solving are specified in
  [`../type-system/checking-rules.md`](../type-system/checking-rules.md), and
  the diagnostics raised on failure are listed in
  [`../type-system/errors.md`](../type-system/errors.md).

## Instantiation and monomorphization

Each distinct assignment of type arguments identifies a separate
*instantiation*. The meaning of a program is defined by an as-if rule:

- A generic declaration behaves **as if** a distinct, non-generic copy were
  written for each instantiation, with every occurrence of the type parameters
  replaced by the corresponding type arguments.
- Instantiations are independent: using `first` at `int` and at `string` means
  exactly what two hand-written functions `first_int` and `first_string` would
  mean.
- Wherever a type parameter occurs, distinct instantiations are distinct types:
  `Pair<int, string>` differs from `Pair<string, int>`, and `list<int>` differs
  from `list<string>`.
- No operation observes a type argument at runtime: no value carries one, no
  expression can request one, and no type pattern may name a type argument or
  a type parameter — a `match` type pattern tests a whole type only
  ([`../expressions/control-flow.md`](../expressions/control-flow.md)). There is therefore
  no observable difference between a program written with generics and the same
  program written with one concrete definition per instantiation. Diagnostics are
  correspondingly per-instantiation: an error in a generic body applies whenever
  the declaration is instantiated at a violating type.

## Generic constraints and dispatch

Inside a body that declares `T: Trait`, values of type `T` expose the members
declared by `Trait`, and the trait's methods may also be invoked explicitly
through the bound.

```xulo
fn describe<T: Area>(shape: T): float {
  shape.area() + Area.area(shape)   // member form and explicit form, one target
}
```

Rules:

- A member call on a bounded type parameter MUST resolve through a bound of that
  parameter; a bound is its only source of members. A value of an *unbounded*
  parameter has no members at all: `t.area()` inside `fn f<T>(t: T)` is an
  error.
- Explicit static dispatch `Trait.method(value)` is available wherever the
  receiver's type is a parameter bounded by `Trait`, and denotes the same call
  as the member form.
- Both forms resolve from the bound written in the declaration. A call whose
  target depends on a value of a type parameter MUST be justified by that static
  syntax; no construct selects a method from the runtime type of a value. Since
  the call site has already checked the argument against the bound, the target is
  known before the argument is chosen.

## Recursive and mutually recursive generics

Generic functions, generic types, and generic declarations that refer to
themselves are allowed, and carry the same termination caveats as ordinary
recursion (see [`../functions.md`](../functions.md)).

```xulo
struct Node<T> { value: T, children: list<Node<T> > }

fn total<T>(node: Node<T>): int {
  let mut sum = 1
  for child in node.children { sum = sum + total(child) }
  sum
}

fn outer<T>(x: T, n: int): int {
  if n == 0 { 0 } else { inner(x, n - 1) }
}

fn inner<T>(x: T, n: int): int {
  if n == 0 { 1 } else { outer(x, n - 1) }
}
```

- A generic type may mention itself in its own type arguments (`Node<T>`
  contains `list<Node<T> >`); that describes finite values and needs no special
  rule. Mutually recursive generic functions may instantiate each other at equal
  or different type arguments; every instantiation is subject to the inference
  and bound rules above.

## Common patterns

**Generic container.** A parameterized `struct` holds a value of that type; an
`impl` MAY carry the same parameter.

```xulo
struct Holder<T> { value: T }

impl<T> Holder<T> { fn get(self): T { self.value } }

let h = Holder(value: 42)   // Holder<int>
let v = h.get()             // int
```

**Identity function.** The smallest generic function: it returns its argument
unchanged.

```xulo
fn identity<T>(x: T): T { x }
```

**Mapping a list.** Two parameters relate the input element type to the output
element type; each is inferred from one argument.

```xulo
fn map<T, U>(xs: list<T>, f: fn(T): U): list<U> {
  let mut out: list<U> = []
  for x in xs {
    out = out + [f(x)]
  }
  out
}

let labels = map([1, 2, 3], fn(n: int): string { str(n) })  // list<string>
```

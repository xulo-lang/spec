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
fn first<T>(xs: List<T>): T { xs[0] }
struct Pair<K, V> { key: K, value: V }
enum Either<L, R> { Left(L), Right(R) }
type ApiResponse<T> = { data: T?, error: String? }

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
- A parameter list has **no default type arguments**: every argument of an
  application is written or inferred; there is no `T = Int` form.
- A type parameter is a type name: it MAY be used wherever a type is expected
  (parameter types, field types, return types, bounds, nested generic
  arguments). It is never a value: a type parameter has no runtime
  representation and never occurs in an expression position. It is also never
  a constructor: writing an application whose head is a type parameter —
  `T<Int>` — is ill-formed (`E0230`), because a parameter stands for a proper
  type, not for a family of them.
- A `trait` is never parameterized. Its grammar accepts a type parameter list
  only so that the attempt can be diagnosed, and declaring one is a
  compile-time error (`E0810`; see [`traits.md`](traits.md)). Trait method
  signatures therefore never mention type parameters.
- Parameter names follow the conventions of [`../names.md`](../names.md): a
  single uppercase letter (`T`, `U`, `K`, `V`) for a simple parameter, or an
  UpperCamelCase descriptive name. These are conventions, not grammar rules.

### Nested angle brackets

`>>` is a single token (the right-shift operator), so two closing angle
brackets MUST NOT be written adjacently. Whenever two `>` characters would
touch, a space MUST be written between them:

```xulo
let nested: List<Map<String, List<Int> > > = []
```

| Form | Well-formed |
|------|-------------|
| `List<List<T> >` | yes |
| `List<List<T>>` | no — `>>` would be read as the shift token |
| `List<List<List<T> > >` | yes |

The rule applies at every generic argument position.

### Type applications

A *type application* is a type written `Name<Args>` — in an annotation, a
field type, a payload, an alias, a parameter or return type, or a pattern's
type. (A bound is not a type application: it never carries type arguments,
under [Bounds](#bounds).) Three rules make every application well-formed:

- The number of type arguments MUST equal the number of type parameters the
  declaration names. `Map<String>` and `Pair<Int, String, Int>` are
  ill-formed (`E0230`).
- A generic declaration's name MUST NOT appear without its arguments:
  `let xs: List = []` is ill-formed (`E0230`), not shorthand for anything.
  A non-generic name carries no argument list at all: `Int<String>` is
  ill-formed (`E0230`) for the same reason.
- A type parameter is never applied (`E0230`); see
  [Type Parameters](#type-parameters).

```xulo
let ok: Map<String, Int> = {}        // well-formed
// let bad: Map<String> = {}         // error[E0230]: `Map` expects 2 type arguments, found 1
// let bare: List = []               // error[E0230]: `List` expects 1 type argument, found 0
```

Type arguments MAY be written in a type position even though they are never
written at a call site ([Inference](#inference)); the two positions differ
only in syntax, not in the checking they receive: written arguments are
unified and bound-checked exactly like inferred ones.

## Bounds

A *bound* restricts the set of type arguments a parameter accepts. Bounds are
written either inline after the parameter name or in a `where` clause that
follows the whole signature.

```xulo
fn area_of<T: Area>(shape: T): Float { Area::area(shape) }
fn perimeter_of<T>(shape: T): Float where T: Area { Area::perimeter(shape) }
```

Rules:

- The forms are equivalent: `<T: Area>` and `where T: Area` constrain the same
  parameter in the same way, and a parameter MUST NOT be bounded in both places.
  A `where` clause follows the parameter list, parameter types, and declared
  return type, and precedes the body; it is a comma-separated list of bounds, one
  per parameter: `fn f<A, B>(x: A, y: B) where A: Area, B: Scalable { … }`.
  A `where` clause MAY follow only a function's signature: `struct`, `enum`,
  `trait`, `type`, and `impl` declarations use the inline form.
- Multiple bounds on one parameter are joined with `+`: `<T: Area + Scalable>`,
  equivalently `where T: Area + Scalable`. The parameter is admissible only if
  its type argument satisfies **every** bound.
- A bound MUST name a `trait` visible at the declaration site; naming anything
  else is a compile-time error (`E0806`)
  (see [`../type-system/errors.md`](../type-system/errors.md)).
- A bound names a trait, never an application of one: writing type arguments on
  a bound — `<T: Convert<String>>` — is a compile-time error (`E0811`). A
  trait is not a type constructor, so there is no such thing as a partially
  or fully applied trait to constrain with.
- A bound is a *constraint*, not a subtyping relation: `T` with bound `Area` does
  not thereby become a subtype of `Area`. Inside the body, values of type `T`
  expose the members declared by `Area`, and explicit dispatch through the bound
  is well-formed ([`traits.md`](traits.md)).
- A bound is never written with `&`: `T & U` is an intersection *type*, which
  combines types a value already has, whereas `T: A + B` restricts which types
  may be substituted for a parameter.

**Bound checking.** Every type argument determined for a declaration MUST
satisfy every bound of the corresponding parameter — whether the argument was
inferred at a call site or written in a type application, and whether the
declaration is a function, a `struct`, an `enum`, an alias, or an `impl`. A
call that would instantiate a parameter with a type that does not implement
one of its bounds is ill-formed, and so is an annotation that names the same
instantiation:

```xulo
trait Area { fn area(self): Float }
struct Palette<T: Area> { shape: T }

let ok = Palette(shape: rect)       // T = Rect, and Rect: Area
// let bad: Palette<Int> = …        // error[E0801]: `Int` does not implement `Area`
```

A violation MUST be reported with a diagnostic even when the type argument was
inferred rather than written; see
[`../type-system/checking-rules.md`](../type-system/checking-rules.md). How a
type is found to satisfy a bound — through which `impl`, and when that `impl`
itself carries bounds — is specified in [`traits.md`](traits.md).

## Inference

A call MAY write the type arguments of a generic callee explicitly, in a list
between the callee and the argument list. The written arguments take the place
of inference for that call:

```xulo
let n = first<Int>([1, 2, 3])      // T = Int, written at the call
let s = first(["a", "b"])          // T = String, inferred as before
```

Rules:

- The list is written `<` … `>` immediately before the argument list,
  comma-separated with an optional trailing comma, and it is **all or
  nothing**: a call writes every type parameter of its callee or writes none.
- The number of type arguments MUST equal the number of type parameters the
  callee declares; a longer or shorter list is `E0808`.
- Each written type argument MUST satisfy every bound of its parameter,
  exactly as an inferred one must; a violation is `E0801`.
- When the list is written, inference is suppressed for those parameters: no
  type variable is introduced for them, no unification constraint is raised
  against the value arguments, and an expected type MUST NOT override a
  written argument. The value arguments are then checked against the
  parameter types with the arguments substituted, and a mismatch is the
  ordinary argument error `E0202`:

  ```xulo
  let a = first<Int>([1, 2, 3])      // OK: `List<Int>` for `List<T>`
  let b = first<Int>(["a", "b"])     // error[E0202]: expected `List<Int>`, found `List<String>`
  ```

- The type of the call is the declared return type with the written
  arguments substituted, and an expected type still checks that result:

  ```xulo
  let xs: Int = first<Int>([1, 2, 3])      // OK: the call has type Int
  let ys: String = first<Int>([1, 2, 3])   // error[E0201]: expected `String`, found `Int`
  ```

- A list written before a callee that declares no type parameters is `E0807`.
- The list is available wherever a call is written — on a function, a method,
  a trait static, or a namespace member — whenever the callee declares type
  parameters:

  ```xulo
  let a = describe<Rectangle>(r)      // T = Rectangle, checked against `T: Area`
  let b = Task.all<Int>([t1, t2])     // T = Int for the built-in namespace member
  ```

- `>>` is a single token, so the nested-bracket spacing rule of this chapter
  applies to a written list as well: `first<Map<String, Int> >(m)` is
  well-formed and `first<Map<String, Int>>(m)` is not.

Whether a `<` after a callee begins such a list is decided by the
disambiguation rule of
[`../expressions/calls.md`](../expressions/calls.md), and the production itself
is in [`../grammar.md`](../grammar.md); this chapter is concerned only with
what a written list means once it is parsed.

**Inference for an omitted list is unchanged.** A call that writes no type
arguments is checked exactly as follows.

- Type arguments are determined by *unification* between the declared parameter
  types and the argument types. Unification decomposes type constructors
  structurally:

  | Constraint | Result |
  |------------|--------|
  | `List<T>` ≟ `List<Int>` | `T = Int` |
  | `Map<K, V>` ≟ `Map<String, Int>` | `K = String`, `V = Int` |
  | `T` ≟ `fn(Int): Boolean` | `T = fn(Int): Boolean` |
  | `T?` ≟ `String?` | `T = String` |
  | `List<T>` ≟ `Map<String, Int>` | fails (different heads) |

- A parameter occurring twice in a signature receives one solution; if two
  constraints demand different types for it, inference fails and the later
  argument is reported as a mismatch against the type the earlier one fixed
  (`E0202`):

  ```xulo
  fn both<T>(a: List<T>, b: List<T>): T { a[0] }
  let t = both([1], ["x"])    // error[E0202]: T is Int from `a`; `List<String>` does not match `List<Int>`
  ```
- An *expected type* MAY determine what the arguments leave undetermined — an
  empty list literal, a numeric literal, a closure with unannotated parameters.
  It MUST NOT override a type already fixed by the arguments: in
  `let bad: String = first([1, 2])`, `T` is `Int`, so the declaration is an
  error rather than an invitation to choose `T = String`. When the list is
  written, the expected type checks the substituted result instead of feeding
  anything back into the arguments.
- Inference MUST fail with a diagnostic when a parameter remains
  under-constrained:

  ```xulo
  fn empty<T>(): List<T> { [] }
  let xs = empty()             // error: T is not determined
  let ys: List<Int> = empty()  // OK: T = Int from the expected type
  ```

  Such a call is repaired by writing the list — `empty<Int>()` — or by
  annotating its context. Constraint generation and solving are specified in
  [`../type-system/checking-rules.md`](../type-system/checking-rules.md), and
  the diagnostics raised on failure are listed in
  [`../type-system/errors.md`](../type-system/errors.md).

The written list is an additional way to supply what inference would
otherwise have to find; it adds no new freedom to a call that omits it.

## Instantiation and monomorphization

Each distinct assignment of type arguments identifies a separate
*instantiation*. The meaning of a program is defined by an as-if rule:

- A generic declaration behaves **as if** a distinct, non-generic copy were
  written for each instantiation, with every occurrence of the type parameters
  replaced by the corresponding type arguments.
- Instantiations are independent: using `first` at `Int` and at `String` means
  exactly what two hand-written functions `first_int` and `first_string` would
  mean.
- Wherever a type parameter occurs, distinct instantiations are distinct types:
  `Pair<Int, String>` differs from `Pair<String, Int>`, and `List<Int>` differs
  from `List<String>`.
- No operation observes a type argument at runtime: no value carries one, no
  expression can request one, and neither a type pattern nor the right operand
  of `is` may name a type argument or a type parameter — both test a whole
  type only
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
fn describe<T: Area>(shape: T): Float {
  shape.area() + Area::area(shape)   // member form and explicit form, one target
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
struct Node<T> { value: T, children: List<Node<T> > }

fn total<T>(node: Node<T>): Int {
  let mut sum = 1
  for child in node.children { sum = sum + total(child) }
  sum
}

fn outer<T>(x: T, n: Int): Int {
  if n == 0 { 0 } else { inner(x, n - 1) }
}

fn inner<T>(x: T, n: Int): Int {
  if n == 0 { 1 } else { outer(x, n - 1) }
}
```

- A generic type may mention itself in its own type arguments (`Node<T>`
  contains `List<Node<T> >`); that describes finite values and needs no special
  rule. Mutually recursive generic functions may instantiate each other at equal
  or different type arguments; every instantiation is subject to the inference
  and bound rules above.

## Common patterns

**Generic container.** A parameterized `struct` holds a value of that type; an
`impl` MAY carry the same parameter.

```xulo
struct Holder<T> { value: T }

impl<T> Holder<T> { fn get(self): T { self.value } }

let h = Holder(value: 42)   // Holder<Int>
let v = h.get()             // Int
```

**Identity function.** The smallest generic function: it returns its argument
unchanged.

```xulo
fn identity<T>(x: T): T { x }
```

**Mapping a list.** Two parameters relate the input element type to the output
element type; each is inferred from one argument.

```xulo
fn map<T, U>(xs: List<T>, f: fn(T): U): List<U> {
  let mut out: List<U> = []
  for x in xs {
    out = out + [f(x)]
  }
  out
}

let labels = map([1, 2, 3], fn(n: Int): String { str(n) })  // List<String>
```

# Traits

A `trait` declares a named set of method signatures — a capability that types
opt into with an `impl`. Traits provide the polymorphic dimension of Xulo:
generic code is written once against a bound, and every call resolves statically
to one implementation. This chapter covers declaring traits, implementing them,
dispatch, and the difference between structural and nominal satisfaction; the
type relations used by bounds are defined in
[`type-relations.md`](type-relations.md).

## Declaration

```xulo
trait Area {
  fn area(self): float;
  fn perimeter(self): float
}
```

Rules:

- A trait declaration is `trait` Identifier `{` methods `}`. Method declarations
  are separated by `;`, and a `;` after the final method is permitted.
- A trait declares **no generic parameters**. Its grammar accepts a generic
  parameter list only so that the attempt can be diagnosed; declaring one —
  `trait Convert<T> { … }` or `trait Sized<const N: int> { … }` — is a
  compile-time error (`E0810`), and a trait's methods therefore never mention
  type or const parameters. A trait is a fixed contract: a type that should
  convert to several targets implements one trait per target, not one
  parameterized trait ([`generics.md`](generics.md)).
- A trait method is a *signature only*: it declares parameters and a return type
  and has no body. A trait MUST NOT provide a default implementation, and an
  implementation therefore cannot inherit code from the trait it implements.
- Every trait method MUST declare a return type. Unlike an ordinary `fn`, whose
  omitted return type means `unit`, a trait method may not omit it.
- The first parameter of a trait method is its *receiver*, written `self` or
  `mut self`. `self` borrows the receiver immutably; `mut self` borrows it
  mutably. Borrowing rules are specified in
  [`../memory-and-runtime.md`](../memory-and-runtime.md). The receiver form
  used by an implementation MUST match the one the trait declares.
- Remaining parameters are ordinary parameters and MUST be annotated with their
  types.
- A trait is a module-level declaration. `pub trait Area { … }` exports the
  trait so that other modules may use it — normally with a type-only binding,
  `use type form::Area;` — and implement it locally; without `pub` the
  trait is private to its module. See
  [`../modules/README.md`](../modules/README.md).
- Trait method signatures are checked for well-formedness when the trait is
  declared; a signature naming an unknown type is reported immediately (see
  [`../type-system/errors.md`](../type-system/errors.md)).

## Implementation

An `impl` block attaches methods to a type. It either implements a trait
(`impl Trait for Type`) or declares inherent methods (`impl Type`).

```xulo
struct Rectangle { w: float, h: float }

impl Area for Rectangle {
  fn area(self): float { self.w * self.h }
  fn perimeter(self): float { 2.0 * (self.w + self.h) }
}

struct User { name: string, age: int }

impl User {
  fn rename(mut self, next: string) {
    self.name = next
  }

  pub fn years(self): int { self.age }
}
```

Rules:

- A trait implementation MUST provide **every** method the trait declares. An
  implementation that omits a method, or whose parameter or return types are not
  mutually assignable with the declared signature, is ill-formed. There are no
  partial implementations and no `default` methods to fall back on.
- A trait implementation MUST NOT declare methods that the trait does not
  declare; additional behaviour belongs in an inherent `impl` block of the type.
- An `impl` block MAY be generic, with its own type and const parameter list
  and bounds on type parameters, and MAY declare inherent methods as well as
  trait methods:

  ```xulo
  struct Pair<A, B> { first: A, second: B }
  struct Tagged<const N: int, T> { value: T }

  impl<T> Pair<T, T> {
    fn duplicated(self): T { self.first }
  }

  impl<const N: int, T> Tagged<N, T> {
    fn size(self): int { N }
  }
  ```

- **When an impl applies.** A generic `impl` applies to a type exactly when
  its type and const parameters can be instantiated so that the implemented
  type — `Pair<T, T>` or `Tagged<N, T>` above — equals that type, and the
  impl's type bounds are satisfied by that same instantiation. Matching the
  implemented type determines the type and const values; this is not a call
  and is not const argument inference at a use site. A const parameter MUST
  occur in the implemented type so matching determines it; an impl whose
  const parameter does not occur is ill-formed (`E0817`). An impl that cannot
  be so instantiated, or whose bounds are not satisfied, contributes
  **nothing** for that type: no members through member-call resolution, and
  no trait implementation for explicit dispatch or bound checking. Two impls
  that can apply to the same type after substitution conflict under the
  ordinary `E0805` rule. Satisfaction is recursive: an impl whose bounds name
  other traits applies only when those bounds hold in turn — which, because
  traits are never parameterized, always means a bound naming a trait with no
  arguments.

  ```xulo
  struct Palette<T> { shape: T }
  struct Rect { w: float, h: float }
  struct Dot { r: float }

  impl Area for Rect {
    fn area(self): float { self.w * self.h }
    fn perimeter(self): float { 2.0 * (self.w + self.h) }
  }

  impl<T: Area> Palette<T> {
    fn render(self): float { Area::area(self.shape) }
  }

  let ok = Palette(shape: Rect(w: 3.0, h: 4.0))
  print(str(ok.render()))          // 12.0: Rect: Area, so the impl applies

  let bad = Palette(shape: Dot(r: 1.0))
  // bad.render()                  // error[E0102]: no member `render` on `Palette<Dot>`
                                   //   the impl's bound `T: Area` is not satisfied
  ```

- Inherent methods are private to the module unless declared `pub`; `pub` on a
  method controls member visibility, exactly as it does for struct fields (see
  [`../README.md`](../README.md)).
- An `impl` applies within the module that contains it. Importing a type or a
  trait does not import any `impl`; a module that needs an implementation
  declares it locally. Two implementations of the same trait for the same type
  MUST NOT be declared in the same module.
- That single-module rule is the whole of the uniqueness rule. A program MAY
  contain several implementations of one trait for one type, one in each module
  that declared one, and `E0805` is raised only when the two live in the same
  module ([`../type-system/errors.md`](../type-system/errors.md)). A call site
  sees the implementations of its own module alone, so no concrete call ever
  has two candidates drawn from two modules, and "exactly one implementation"
  below is always a statement about one module's view.

## Dispatch

Dispatch in Xulo is explicit and static: every call names, or is resolved from
a bound that names, exactly one implementation.

### Explicit dispatch

```xulo
let r = Rectangle(w: 3.0, h: 4.0)
let a = Area::area(r)
let p = Area::perimeter(r)
```

- `Trait.method(receiver, …)` resolves directly to the method of the `impl`
  that implements `Trait` for the static type of the receiver. The receiver
  MUST resolve to a concrete named type with a matching `impl` visible at the
  call site; otherwise the call is an error (`E0801`), reported as "does not
  implement trait" (see [`../type-system/errors.md`](../type-system/errors.md)).
- No runtime type information is consulted: the target is fixed before the
  program runs.

### Member calls

A call written `value.method(…)` is resolved in a fixed order:

1. methods declared by inherent `impl` blocks of the receiver's type that are
   visible in the current module;
2. otherwise, methods declared by traits that are in scope, implemented for the
   receiver's type, and visible in the current module.

Exactly one candidate MUST result. If step 1 finds no method and step 2 finds
none, the call is an error (`E0102`); if step 2 finds more than one candidate,
the call is an ambiguity error (`E0809`) and MUST be disambiguated with
explicit dispatch `Trait.method(receiver)`.

Beyond the two steps — when neither finds a candidate — a receiver whose
type implements `Display`, through its own type or through a `T: Display`
bound, resolves `to_string(…)` to the derived action of that impl: the
member is found without appearing in any trait's declared members,
dispatches to `display` at the default `Format`, and yields to anything
either step found (an inherent `to_string` wins at step 1, one declared by
another in-scope trait at step 2). An `impl Display` that declares
`fn to_string` remains `E0804`: the action is provided, not declared.

There is no implicit method resolution through an unbounded generic parameter:
for `fn f<T>(x: T)`, `x.method()` is an error (`E0102`), because a type parameter has no
members unless a bound grants them. See [`generics.md`](generics.md).

## Bounds and generic code

A generic declaration constrains its parameters with `<T: Area>` or
`where T: Area`; both forms, multiple bounds, checking at every position a
type argument is written or inferred, and the rule that a bound never carries
type arguments are specified in [`generics.md`](generics.md).

A type satisfies a bound exactly when a visible `impl` for the bound's trait
applies to it, under the applicability rules above — including the recursion
through an impl's own bounds. That check is the whole of bound satisfaction:
there is no structural fallback and no second source of evidence.

Inside a body that carries the bound, values of type `T` expose the members the
bound declares, and either spelling denotes the same static call:

```xulo
fn area_of<T: Area>(shape: T): float {
  Area::area(shape)      // explicit static dispatch
}

fn area_twice<T: Area>(shape: T): float {
  shape.area() + shape.area()   // member form, resolves through the bound
}
```

The language rule for calls that depend on a type parameter is:

- A call whose target depends on a value of a type parameter MUST be justified
  statically by a bound written in the declaration — either through the member
  form, which resolves through the bound, or through explicit dispatch
  `Trait.method(value)`.
- No expression performs a method lookup on the runtime type of a value. There
  is no dynamic dispatch, no default method fallback, and no reflection over
  type arguments.
- Consequently, a bound is checked once per instantiation, at the point that
  selects the type argument — a call that infers it, or a type application
  that writes it — and every call inside the body is then fixed.
- **Which implementation a bound uses.** The implementation that satisfies a
  bound is the one visible at the point that selects the type argument, and it
  travels with the bound into the generic body: every bound-resolved call
  there — `Area::area(value)` or `value.area()` alike — uses that same
  implementation for that instantiation. The module in which the generic was
  declared takes no part in the choice, so a generic written in a module that
  declares no implementation for the trait is usable wherever an instantiation
  point supplies one. Two instantiations of the same generic with the same type
  argument at call sites in different modules MAY therefore use two different
  implementations; that is the direct consequence of module-scoped `impl`s and
  is specified behavior, not an ambiguity. An explicit `Trait.method(x)` on a
  value whose type is already concrete, written outside a generic body,
  continues to use the implementation visible at that call site.

## Structural vs nominal

- A type satisfies a trait **only through an `impl`**. A type that happens to
  declare methods with the right names and signature does not implement the
  trait; satisfaction is never inferred from the shape of a type.
- `Map` and tuple types are structural for *assignability* — they depend only
  on their element types — while `struct` types are nominal: a `struct` is
  assignable only to its own type. In both cases, structural compatibility
  never implies trait satisfaction (see
  [`type-relations.md`](type-relations.md)).
- A `type` alias is transparent: it introduces no new type, and it never, by
  itself, causes a type to implement a trait. Implementations attach to the
  type an alias expands to, never to the alias itself (see
  [`composite-types.md`](composite-types.md)).
- `Display`, `Debug`, and `Format` are built-in; no declaration introduces
  them, and the string-form rule, the derived `to_string`, and the formatting
  contracts are stated in
  [Built-in formatting traits](#built-in-formatting-traits).

## Built-in formatting traits

`Display`, `Debug`, and the struct `Format` are built-in: the prelude
provides them, no declaration introduces them, and every module sees them
([`../builtins/prelude.md`](../builtins/prelude.md#built-in-protocols)).
Their declarations, as the prelude states them:

```xulo
struct Format {
  precision: int?,   // the precision a directive asked for; `null` = none
                     // asked — `0` is a request, not an absence
  alternate: bool    // the `#` alternate form; `false` = not asked
}

trait Display {
  fn display(self, format: Format): string
}

trait Debug {
  fn debug(self, format: Format): string
}
```

- `Display` is **how a type is shown**: `display` returns the receiver's
  string form at the `Format` it is given. Every consumer that renders with
  no directive — interpolation in `` `…${…}…` ``, the intrinsic `str` (and
  a member that spells its `str` domain, `List.join`), `print`, and the
  derived `to_string` — passes the **default** `Format`,
  `Format(precision: null, alternate: false)`.
- `Format` carries **semantics only**. It has no width, alignment, fill, or
  padding field: layout is applied by the consumer to the string `display`
  returns, never by an implementation, so a rendering request is honored
  whether or not the implementation looked at `format`. A `display` that has
  a meaning for `format.precision` or `format.alternate` SHOULD reflect it;
  ignoring either still leaves a well-formed rendering, because whatever the
  type did not lay out, the consumer does. A consumer hands a directive's
  precision to `display` unchanged: the contract fixes the field, not its
  unit — what the number counts (decimal places, significant digits,
  characters) is the implementation's reading of its own type.
- The field set may grow, but only by fields of an optional type: a `T?`
  field MAY be omitted at construction and takes `null`
  ([`../types/composite-types.md`](../types/composite-types.md)), so a
  construction written against the present fields keeps compiling and keeps
  its meaning — an omitted field always reads as "the consumer asked for
  nothing", the default rendering. A future boolean flag is written `bool?`
  for the same reason: the contract declares its `null` and its `false` one
  reading — *not asked* — because a non-optional field would fail every
  existing construction (`E0205`).
- `Debug` is **what a type shows of itself**: `debug` returns the structural
  form — the value's internals rendered the way the output table renders
  structure
  ([`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md#output)).
  The structural form does not read `format` in this version; the parameter
  is part of the signature so that a later rendering can.
- Every `Display` impl makes the **action** available: `value.to_string()` is
  `value.display` at the default `Format`. The language provides the method
  with the impl — it dispatches through that impl, is resolved wherever the
  impl is visible (see [Member calls](#member-calls)), and is declared by
  no impl: an `impl Display` that declares
  `fn to_string` is `E0804`, because `Display` declares no such method.
- A value interpolated by `${…}`, passed to `str`, or rendered through a
  member that spells the `str` domain (`List.join`'s elements) MUST be a
  base type (`int`, `float`, `bool`, `string`) or a type that implements
  `Display`; other types are a compile-time error in those positions
  (`E0212`).

  ```xulo
  struct User { name: string, age: int }

  impl Display for User {
    fn display(self, format: Format): string {
      `${self.name} (${self.age})`
    }
  }
  ```

## Attributes

A declaration MAY carry attributes written directly before it: the form
`#[name]`, optionally with a parenthesized argument list inside the brackets;
what the arguments may contain is fixed per entry by the registry —
`#[derive]` takes a comma-separated list of trait names with an optional
trailing comma.

```xulo
#[derive(Debug)]
struct User { name: string, age: int }
```

An attribute on its own line, directly above the declaration it marks, is
**RECOMMENDED**; a shared line — `#[derive(Debug)] struct User { … }` — is
equally well-formed, the grammar being newline-insensitive
([lexical-structure.md](../lexical-structure.md)).

- The attribute form is generic; the names the checker interprets form a
  **closed registry**. An attribute whose name is not in the registry is
  `E0108`. There are no inert attributes: every attribute written in source
  is interpreted by the checker.
- Attribute names occupy their own namespace. `#[derive]` binds nothing and
  takes no name away from values, types, or modules.
- A registered attribute is valid only on the declarations its registry entry
  allows (`E0109`), and its arguments must fit the shape that entry states
  (`E0110`). Attributes are written on the five item declarations — `fn`,
  `struct`, `enum`, `trait`, and `impl` — and on the test form (`#[test]`),
  and on nothing else
  ([`../grammar.md`](../grammar.md)).
- The registry of derive targets is closed in the same way. The only
  registered target is `Debug`. `Display` and `ToString` are reserved names
  in this registry and are not registered; written as targets they are
  `E0111`. `Display` is never a derive target: the user-visible string form
  is an author's decision
  ([Built-in formatting traits](#built-in-formatting-traits)).

### Derive

`#[derive(Debug)]` on a `struct` or an `enum` generates exactly the impl an
author would write by hand for the structural case. The body is fixed by the
target:

- On a struct `S` with fields `f₁ … fₙ`, the generated `impl Debug for S`
  renders the value as `S(f1: …, f2: …)` — the fields in declaration order,
  each rendered by the field rules below: directly where no implementation
  is needed, through `debug` where one is. A struct with no fields renders
  as `S()`.
- On an enum, the generated body renders each variant exactly as the output
  table renders an enum value — `E::V`, or `E::V(p, …)` — with every payload
  value rendered by the field rules above. Payload slots are positional
  whether the variant declares them positionally or by name
  ([`../types/enums.md`](../types/enums.md)).
- The generated `debug` takes its `format` parameter like every `Debug`
  method: it forwards `format` to the `debug` calls it makes on
  user-declared fields and has no other use for it — the structural form is
  fixed. A body (generated or written) that never calls `debug` simply does
  not read the parameter, and a parameter is never reported as unused (the
  note under the warnings table of
  [`../type-system/errors.md`](../type-system/errors.md)).

```xulo
#[derive(Debug)]
struct User { name: string, age: int }

#[derive(Debug)]
enum Shape { Circle(float), Rect(float, float) }

let s = User(name: "Alice", age: 30)
print(s)                          // User(name: Alice, age: 30)
print(Shape::Circle(2.5))         // Shape::Circle(2.5)
```

Rules:

- The rendering of a field or payload is chosen by its type:
  - a `struct` renders through its `debug`; without a `Debug` impl the
    derive fails with `E0801` at that type: `` `User` does not implement
    `Debug` `` — a struct never renders directly;
  - an `enum` renders through its `debug` when it implements one, and
    otherwise by the output table's enum row, which needs no
    implementation — the same fallback `print` makes;
  - any other type renders directly — no implementation required **of the
    type itself** — by the output table's rules for its kind (a base type
    as itself, a container from its elements), the recursion below
    carrying these rules into every structural position that rendering
    reaches;
  - a value typed `unknown` — in a field or in any structural position —
    has no static rendering to expand: the derive fails with `E0212`, the
    error the same value's interpolation already gets.
- The rules are recursive through **structural positions** — a container's
  elements, an optional's `T`, a tuple's elements, a `Result`'s `T` and
  `E` slots, an enum's payloads (the term is defined with the output table
  ([`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md#output))) —
  and every position is rendered, and checked, by these same rules:
  `List<User>`, `User?`, `(User, int)`, and `Result<User, string>` each
  need `User`'s `Debug` and fail `E0801` at `User` without it. A wrapper is
  itself a user-declared type: `struct W { xs: List<int> }` derives because
  its field renders directly, and a `W` used as another derive's field
  needs `W`'s own `Debug`.
- The expansion is an ordinary impl, so a hand-written `impl Debug` for
  the same type, or a second `#[derive(Debug)]`, is the conflicting-impl
  error `E0805`.
- `#[derive]` on a type that declares generic parameters is `E0112`: the
  language has no bound that means "is not a `struct`, or implements
  `Debug`", so the expansion cannot be written for an open
  parameter. A generic type keeps its hand-written impl.
- A derived impl participates in dispatch exactly as a written one; derive
  introduces no satisfaction that an `impl` would not.
- `#[derive(Display)]` is `E0111`: user-visible text is written by hand, and
  the structural form a derive could default to is `Debug`'s.

### Test

`#[test]` marks a function as a test. It is spelled only by the test form —
`#[test]`, followed by `[async] fn name() { … }` — whose shape is fixed
([`../testing.md`](../testing.md)):

- the declaration has no parameters, no return type, no generic parameters,
  no `where` clause, and no `pub`;
- the attribute takes no arguments: `#[test()]`, `#[test("…")]`, and
  `#[test(name, …)]` do not form a `TestDecl`;
- the identifier binds no value, and any reference to it is `E0107`.

Written on a `struct`, `enum`, `trait`, or `impl`, `#[test]` is `E0109`
(`` attribute `test` is not valid on a struct ``). When the full test form
does not match, a `#[test]`, `#[test()]`, or `#[test(name, …)]` on the
non-matching `fn` — including a `fn` with parameters, a return type,
generics, a `where` clause, or `pub` — is `E0110`
(`` attribute `test` takes no arguments and is written `#[test]`, followed
by `[async] fn name() { … }` ``). A `#[test(...)]` group that the current
`AttributeArg` grammar cannot parse — notably a `StringLiteral` or a numeral
— is a syntax error when the full test form does not match; when
`add-macro-system` widens `AttributeArg`, such a group parses and falls under
the same `E0110` rule. When the sequence `#[test]` followed by `async`? `fn`
Identifier `(` `)` Block appears at file scope, the declaration is a
`TestDecl` and no registry lookup runs ([`../grammar.md`](../grammar.md)).

## Example

A complete example: one trait, two structs, a generic function bounded by the
trait, and explicit dispatch.

```xulo
trait Area {
  fn area(self): float;
  fn perimeter(self): float
}

struct Rectangle { w: float, h: float }
struct Circle { r: float }

impl Area for Rectangle {
  fn area(self): float { self.w * self.h }
  fn perimeter(self): float { 2.0 * (self.w + self.h) }
}

impl Area for Circle {
  fn area(self): float { 3.14159 * self.r * self.r }
  fn perimeter(self): float { 2.0 * 3.14159 * self.r }
}

fn describe<T: Area>(shape: T): float {
  Area::area(shape) + Area::perimeter(shape)
}

fn main() {
  let r = Rectangle(w: 3.0, h: 4.0)
  let c = Circle(r: 2.0)
  print(str(Area::area(r)))
  print(str(Area::perimeter(c)))
  print(str(describe(r)))
  print(str(describe(c)))
}
```

`describe` is checked once for `T: Area`; the calls `describe(r)` and
`describe(c)` infer `T = Rectangle` and `T = Circle`, and each is valid only
because a visible `impl Area for …` exists for that type.

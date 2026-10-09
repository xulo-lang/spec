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
  fn area(self): Float;
  fn perimeter(self): Float
}
```

Rules:

- A trait declaration is `trait` Identifier `{` methods `}`. Method declarations
  are separated by `;`, and a `;` after the final method is permitted.
- A trait method is a *signature only*: it declares parameters and a return type
  and has no body. A trait MUST NOT provide a default implementation, and an
  implementation therefore cannot inherit code from the trait it implements.
- Every trait method MUST declare a return type. Unlike an ordinary `fn`, whose
  omitted return type means `Unit`, a trait method may not omit it.
- The first parameter of a trait method is its *receiver*, written `self` or
  `mut self`. `self` borrows the receiver immutably; `mut self` borrows it
  mutably. Borrowing rules are specified in
  [`../memory-and-runtime.md`](../memory-and-runtime.md). The receiver form
  used by an implementation MUST match the one the trait declares.
- Remaining parameters are ordinary parameters and MUST be annotated with their
  types.
- A trait is a module-level declaration. `pub trait Area { … }` exports the
  trait so that other modules may import it — normally with
  `import type { Area } from "…"` — and implement it locally; without `pub` the
  trait is private to its module. See
  [`../modules/README.md`](../modules/README.md).
- Trait method signatures are checked for well-formedness when the trait is
  declared; a signature naming an unknown type is reported immediately (see
  [`../type-system/errors.md`](../type-system/errors.md)).

## Implementation

An `impl` block attaches methods to a type. It either implements a trait
(`impl Trait for Type`) or declares inherent methods (`impl Type`).

```xulo
struct Rectangle { w: Float, h: Float }

impl Area for Rectangle {
  fn area(self): Float { self.w * self.h }
  fn perimeter(self): Float { 2.0 * (self.w + self.h) }
}

struct User { name: String, age: Int }

impl User {
  fn rename(mut self, next: String) {
    self.name = next
  }

  pub fn years(self): Int { self.age }
}
```

Rules:

- A trait implementation MUST provide **every** method the trait declares. An
  implementation that omits a method, or whose parameter or return types are not
  mutually assignable with the declared signature, is ill-formed. There are no
  partial implementations and no `default` methods to fall back on.
- A trait implementation MUST NOT declare methods that the trait does not
  declare; additional behaviour belongs in an inherent `impl` block of the type.
- An `impl` block MAY be generic, with its own type parameter list and bounds,
  and MAY declare inherent methods as well as trait methods:

  ```xulo
  struct Pair<A, B> { first: A, second: B }
  struct Palette<T> { shape: T }

  impl<T> Pair<T, T> {
    fn duplicated(self): T { self.first }
  }

  impl<T: Area + Scalable> Palette<T> {
    fn render(self): Float { Area.area(self.shape) }
  }
  ```

- Inherent methods are private to the module unless declared `pub`; `pub` on a
  method controls member visibility, exactly as it does for struct fields (see
  [`../README.md`](../README.md)).
- An `impl` applies within the module that contains it. Importing a type or a
  trait does not import any `impl`; a module that needs an implementation
  declares it locally. Two implementations of the same trait for the same type
  MUST NOT be declared in the same module.

## Dispatch

Dispatch in Xulo is explicit and static: every call names, or is resolved from
a bound that names, exactly one implementation.

### Explicit dispatch

```xulo
let r = Rectangle(w: 3.0, h: 4.0)
let a = Area.area(r)
let p = Area.perimeter(r)
```

- `Trait.method(receiver, …)` resolves directly to the method of the `impl`
  that implements `Trait` for the static type of the receiver. The receiver
  MUST resolve to a concrete named type with a matching `impl` visible at the
  call site; otherwise the call is an error ("does not implement trait"), see
  [`../type-system/errors.md`](../type-system/errors.md).
- No runtime type information is consulted: the target is fixed before the
  program runs.

### Member calls

A call written `value.method(…)` is resolved in a fixed order:

1. methods declared by inherent `impl` blocks of the receiver's type that are
   visible in the current module;
2. otherwise, methods declared by traits that are in scope, implemented for the
   receiver's type, and visible in the current module.

Exactly one candidate MUST result. If step 1 finds no method and step 2 finds
none, the call is an error; if step 2 finds more than one candidate, the call is
an ambiguity error and MUST be disambiguated with explicit dispatch
`Trait.method(receiver)`.

There is no implicit method resolution through an unbounded generic parameter:
for `fn f<T>(x: T)`, `x.method()` is an error, because a type parameter has no
members unless a bound grants them. See [`generics.md`](generics.md).

## Bounds and generic code

A generic declaration constrains its parameters with `<T: Area>` or
`where T: Area`; both forms, multiple bounds, and call-site checking are
specified in [`generics.md`](generics.md).

Inside a body that carries the bound, values of type `T` expose the members the
bound declares, and either spelling denotes the same static call:

```xulo
fn area_of<T: Area>(shape: T): Float {
  Area.area(shape)      // explicit static dispatch
}

fn area_twice<T: Area>(shape: T): Float {
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
- Consequently, a bound is checked once per instantiation, at the call site
  that selects the type argument, and every call inside the body is then fixed.

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
- `ToString` is a built-in trait. A value interpolated in a template literal
  `` `…${expr}…` `` must be a base type (`Int`, `Float`, `Boolean`, `String`) or
  implement `ToString`; other types are a compile-time error in that position.

  ```xulo
  struct User { name: String, age: Int }

  impl ToString for User {
    fn to_string(self): String {
      `${self.name} (${self.age})`
    }
  }
  ```

## Example

A complete example: one trait, two structs, a generic function bounded by the
trait, and explicit dispatch.

```xulo
trait Area {
  fn area(self): Float;
  fn perimeter(self): Float
}

struct Rectangle { w: Float, h: Float }
struct Circle { r: Float }

impl Area for Rectangle {
  fn area(self): Float { self.w * self.h }
  fn perimeter(self): Float { 2.0 * (self.w + self.h) }
}

impl Area for Circle {
  fn area(self): Float { 3.14159 * self.r * self.r }
  fn perimeter(self): Float { 2.0 * 3.14159 * self.r }
}

fn describe<T: Area>(shape: T): Float {
  Area.area(shape) + Area.perimeter(shape)
}

fn main() {
  let r = Rectangle(w: 3.0, h: 4.0)
  let c = Circle(r: 2.0)
  print(str(Area.area(r)))
  print(str(Area.perimeter(c)))
  print(str(describe(r)))
  print(str(describe(c)))
}
```

`describe` is checked once for `T: Area`; the calls `describe(r)` and
`describe(c)` infer `T = Rectangle` and `T = Circle`, and each is valid only
because a visible `impl Area for …` exists for that type.

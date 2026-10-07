# Components

A **component** is a function whose declared return type is `View`. Components
are how a Xulo program describes the interface it presents: each component
returns a value describing a fragment of the render tree, and composing
components composes the tree *as data*. No component measures, positions, or
paints anything itself — the tree that components build is measured, positioned,
and painted under the contract specified in [`layout.md`](layout.md). This
chapter specifies the component layer: how components are declared and invoked,
what may appear as their children, and the four declarations — `@State`,
`@Store`, `@Effect`, `@Environment` — that give a component memory, work, and
access to its surroundings.

## Component Functions

A component is declared with `fn` like any other function; the declared return
type `View` is the only thing that makes it a component.

```xulo
fn Greeting(name: string): View {
  Text(`Hello, ${name}`)
}
```

- Component names are conventionally UpperCamelCase (`Counter`, `UserCard`).
  The convention guides readers but is not a grammar rule: componenthood
  follows from the declared return type, never from the spelling of the name
  (see [`../names.md`](../names.md)).
- Component parameters are ordinary typed parameters — required, optional
  (`T?`), and default parameters, positional and named arguments, and generic
  parameters all follow the ordinary rules ([`../functions.md`](../functions.md),
  [`../expressions/calls.md`](../expressions/calls.md)).
- In the type system a component is an ordinary function. `View` is a built-in
  type ([`../types/primitive-types.md`](../types/primitive-types.md)), a
  component invocation is an ordinary call, and a component MAY be evaluated
  wherever a value of type `View` is accepted.
- The entry point `main` determines how a program runs: `fn main(): View` is a
  renderable program whose returned tree is laid out and painted, while a plain
  `fn main()` runs headlessly and paints nothing
  ([`../modules/source-files.md`](../modules/source-files.md)).

## Component Blocks

A component invocation may be followed by a trailing `{ … }` block whose items
are the component's **children**:

```xulo
VStack(spacing: 8) {
  Text("Account balance")
  Button("Deposit", onClick: fn() { deposit() })
}
```

At a glance:

```text
Name(args)              // invocation with arguments, no children
Name(args) { children } // invocation with both
Name { children }       // invocation with children, no arguments
```

The items of a block are component invocations, `if` and `for` control flow,
bare strings (text nodes), and expressions of type `View`, `string`, or
`list<View>` — anything else in a block is a compile-time error. Arguments in
`( )` configure the component; children in `{ }` are the `View`s it embeds.
Blocks, nesting, flattening, and expression children are specified in
[`view-syntax.md`](view-syntax.md).

## Declarations and Attributes

Four declarations are specific to component bodies:

```xulo
fn Counter(): View {
  @State let count: int = 0
  @Effect fn() { print("mounted") }

  VStack {
    Text(`count = ${count}`)
    Button("More", onClick: fn() { count = count + 1 })
  }
}
```

- `@State`, `@Store`, `@Effect`, and `@Environment` are valid **only** at the
  top level of a component body. Writing any of them in an ordinary function,
  in an `async` function, or inside a nested block — an `if`, a `for`, or a
  component block — is a compile-time error
  ([`../type-system/errors.md`](../type-system/errors.md)). This rule is
  normative and has no exceptions.
- Attributes written in `( )` are ordinary named arguments. Which attributes a
  component accepts, and what each one means, is that component's contract; the
  language specifies only how arguments are written, matched, and typed
  ([`view-syntax.md`](view-syntax.md)).
- The `$` prefix in an argument position passes a binding to a state variable
  of the enclosing component instead of a copy of its value; see
  [`binding.md`](binding.md).

## Index

- [`view-syntax.md`](view-syntax.md) — the `View` type, component declarations and invocations, children, and well-formedness.
- [`state.md`](state.md) — `@State` and `@Store`: declaration, updates, identity across renders.
- [`effect.md`](effect.md) — `@Effect`: run rules, dependency lists, cleanup, ordering.
- [`environment.md`](environment.md) — `@Environment`: provisions, lookups, scope and lifetime.
- [`binding.md`](binding.md) — `$` two-way binding, one-way properties, and event handlers.
- [`layout.md`](layout.md) — the rendering contract: layout model, size and style attributes, hit testing, and the render pipeline.

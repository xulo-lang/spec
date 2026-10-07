# Environment

Some values a component needs come neither from its parameters nor from its own
state: a router, a host capability, a session-wide service. `@Environment`
declares a binding to such a value — the binding has no initializer, because
the value is provided from outside the program's own text. This chapter
specifies the declaration form, how provisions are looked up, where provisions
come from, how long they live, and where the declaration is legal.

## `@Environment`

```xulo
fn Nav(): View {
  @Environment let router: Router

  VStack {
    Button("About", onClick: fn() { router.push("/about") })
  }
}
```

- The form is `@Environment let name: T`. It has **no initializer**: the value
  is provided by an enclosing context, never computed in the component.
- The type annotation is **REQUIRED**. Nothing in the declaration supplies the
  type, and no inference can recover it, so `@Environment let router` without
  `: Router` is a compile-time error.
- The declared name binds like an ordinary binding: it reads like a variable in
  the rest of the body, it is captured by closures normally, and it draws from
  the component body's single name pool with `@State` and `@Store`
  ([`../names.md`](../names.md)).
- The type MAY be any type; in practice it is a `struct`, a `trait`, or a type
  alias supplied by a library. The language contributes only the lookup rule,
  not the vocabulary of provision types.

## Provisions and lookups

A **provision** associates a **key** — a type — with a value. The declaration
`@Environment let router: Router` reads the provision whose key is `Router`.

- A value is provided at a root or at an ancestor position of the tree and read
  by components at descendant positions: provisions are established outside-in,
  and a component never establishes a provision for itself.
- **The nearest enclosing provision wins.** Lookup starts at the component's
  position and walks outward; the first position that provides the key supplies
  the value. A provision established further out for the same key is shadowed,
  not merged, and provisions for other keys are unaffected.
- **Reading a key that was never provided is a runtime error.** If no
  enclosing position provides the key, the read of the `@Environment` binding
  fails when it is evaluated — it is not `null`, it is not an empty value, and
  the program does not continue with a partial environment.
- A provision supplies one value for its key: every component that reads that
  key at a position inside the provision observes the same value.

```text
session root      provides Router        ← embedding context
└── Screen
    └── Panel     reads Router           ← nearest provision: the root's
        └── Card  reads Router           ← same provision again
```

## Injection points

Values are supplied by the **embedding context**: before `main` runs, the host
that is running the program establishes the provisions for the render session —
each key with its value — and only then invokes `main`, whose returned tree is
laid out and painted ([`view-syntax.md`](view-syntax.md),
[`layout.md`](layout.md)). Nothing in a `.xulo` source file creates or replaces
a provision; the language defines no construct that establishes one, and a
program's only contact with the environment is reading it with `@Environment`.
Where an embedding establishes provisions at more than one position — an inner
session established within an outer one — the provisions at the innermost
enclosing position are the ones observed by components inside it. The router
example above is the typical case: the host provides a `Router` for the whole
session, and any component anywhere in the tree that declares
`@Environment let router: Router` navigates through it.

## Scope and lifetime

- Environment values live for the **whole session**. A provision is
  established before `main` runs, remains in effect while the program renders
  and re-runs, and is released only when the session ends.
- A component observes the environment **at its position in the tree**. The
  value a component reads is the one visible from where that component sits;
  moving a component to a different position can therefore change which
  provision it observes, while re-running the same component at the same
  position reads the same value again.
- Provisions are not state: reading an `@Environment` binding never schedules a
  re-run, and no assignment to it exists — the binding is read-only like an
  ordinary `let` ([`state.md`](state.md)).

## Legality

`@Environment` is valid **only** at the top level of a component body — as a
direct item of the body of a function whose declared return type is `View`.
Writing it in an ordinary function, in an `async` function, in a closure, or
inside a nested block is a compile-time error
([`../type-system/errors.md`](../type-system/errors.md)). This rule is
normative and has no exceptions.

| Construct | Condition | Result |
|-----------|-----------|--------|
| `@Environment` | Written outside the top level of a component body | error |
| `@Environment` | Type annotation omitted | error |
| `@Environment` | Initializer supplied | error |
| `@Environment` | Name already bound by another declaration in the same body | error |
| Provision | Key read at a position where no enclosing provision provides it | runtime error |

## Example

A component may read an environment value and a store side by side; the two
serve different purposes — the router is supplied from outside, the theme is
shared program state:

```xulo
import type { Router } from "@xulo/router"

fn NavBar(): View {
  @Environment let router: Router
  @Store let { theme, setTheme } = useAppStore()

  HStack(spacing: 8) {
    Button("About", onClick: fn() { router.push("/about") })
    Button("Docs", onClick: fn() { router.push("/docs") })
    if theme == Theme::Dark {
      Text("Dark mode")
    }
    Button("Dark", onClick: fn() { setTheme(Theme::Dark) })
  }
}
```

The router comes from the embedding context and is the same for every reader in
the session; the theme comes from the store and changes over the session,
re-running every component that reads it ([`state.md`](state.md)).

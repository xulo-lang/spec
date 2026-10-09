# State

A component's run is a description of a tree; state is what makes that
description change over time. Xulo has two state declarations — `@State` for
state owned by one component instance and `@Store` for state shared
program-wide — plus `@Effect` ([`effect.md`](effect.md)) for work that happens
outside a run and `@Environment` ([`environment.md`](environment.md)) for
values supplied from outside. This chapter specifies what a state declaration
introduces, how it is read and written, how it identifies itself across runs,
and where the four declarations are legal.

## `@State`

`@State` declares a reactive binding at the top level of a component body.

```xulo
fn Counter(): View {
  @State let count: Int = 0

  VStack(spacing: 8) {
    Text(`count = ${count}`)
    Button("More", onClick: fn() { count = count + 1 })
    Button("Reset", onClick: fn() { count = 0 })
  }
}
```

- The form is `@State let name: T = expression`, or `@State let name =
  expression` when the type is inferable.
- The type annotation is optional when the initializer determines the type:
  `@State let count = 0` declares `count: Int`. When the initializer does not
  determine the type — an empty list literal, for example — the annotation is
  REQUIRED ([`../statements/let-and-assignment.md`](../statements/let-and-assignment.md)).
- Once declared, the binding reads exactly like an ordinary variable: `count`
  in an expression is its current value, and nothing about the `@` shows
  through at a use site.
- The initializer is evaluated when the component instance is created, not on
  every run of the body. A later run re-reads the retained value; it does not
  re-evaluate the initializer (see *Identity across renders* below).
- `@State` bindings are **implicitly mutable**: unlike an ordinary `let`, they
  MAY be assigned, and they are assigned in exactly the same way — `count =
  count + 1` in the body or in a handler created there. The ordinary rule that
  assigning to a non-`mut` binding is a compile-time error
  ([`../statements/let-and-assignment.md`](../statements/let-and-assignment.md))
  applies to every binding except one introduced by `@State`, whose purpose is
  to be written. No `mut` is written, and there is no form `@State let mut`.

## Signals and updates

For the purpose of rendering, an `@State` binding behaves as a signal:

- **Reading** the binding during a run yields the current value of that state
  cell — the value most recently written.
- **Assigning** to the binding writes the cell and **schedules the component to
  re-run**. The body is executed again once the assignment is processed, and
  that run reads all of the component's `@State` bindings afresh.
- Because the body runs again, everything the run derives from state — text
  built from a template, children chosen by `if`, children repeated by `for` —
  is recomputed. Derived values are not stored; they are recomputed on each
  run.
- An assignment may occur wherever a statement may: in an event handler
  ([`binding.md`](binding.md)), in an `@Effect` body, or in ordinary code that
  runs while the program is live.

```xulo
fn Counter(): View {
  @State let count: Int = 0

  HStack(spacing: 4) {
    Button("-", onClick: fn() { count = count - 1 })
    Text(str(count))
    Button("+", onClick: fn() { count = count + 1 })
  }
}
```

## Identity across renders

State belongs to a **component instance**, not to a component's code:

- An `@State` value is **retained across re-runs** of the same component
  instance. Re-running the body does not reset it to its initializer.
- A state cell is **created fresh for each new instance**. When a component is
  instantiated anew — at a position it did not occupy before, or after having
  been removed from the tree and rendered again — its `@State` initializers run
  again for that instance.
- **Two instances of the same component MUST NOT share state.** Two invocations
  of `Counter` in the same parent produce two independent counts, because they
  are two instances with two sets of state cells. This is the specified
  semantics of `@State`.

```xulo
fn main(): View {
  VStack {
    Counter()   // first instance, its own `count`
    Counter()   // second instance, its own `count`
  }
}
```

An instance is identified by its position in the render tree — the chain of
child positions that leads to it — and each of an instance's `@State`
declarations addresses one cell of that instance. A run of the component
re-associates the declarations with the cells of the instance being run; a run
of a different instance addresses that instance's cells.

## `@Store`

`@Store` declares bindings that read a program-wide store. Its form is a
destructuring binding, because a store is consumed by its fields:

```xulo
fn Home(): View {
  @Store let { user, theme } = useAppStore()
  @Store let { setTheme } = useAppStore()

  VStack {
    Text(`signed in as ${user?.name ?? "guest"}`)
    if theme == Theme::Dark {
      Text("Dark mode")
    }
    Button("Toggle theme", onClick: fn() { setTheme(Theme::Dark) })
  }
}
```

- The names inside `{ … }` are bound exactly as if written with `@State`: they
  are ordinary names in the component body, they read like ordinary variables,
  and they draw from the same per-body name pool as `@State` and
  `@Environment` ([`../names.md`](../names.md)).
- A store binding may name values (`user`, `theme`) or functions the store
  exposes (`setTheme`); both come from the same declaration form, and a body
  MAY contain several `@Store` declarations.
- **Store values are shared.** Every component that declares them observes the
  same values: there is one store, and its state is not per instance. State that
  must be common to the program — a theme, a current user — belongs in a store;
  state that must be per instance belongs in `@State`.
- **Updating the store re-runs every component that reads it.** An update
  through a store's setters (or through a `$` binding, [`binding.md`](binding.md))
  schedules a re-run of every component that declared `@Store` for that store;
  each of those components re-reads the store, and their own `@State` cells are
  untouched. Components that read no part of the store are not re-run.

## Where declarations are legal

`@State`, `@Store`, `@Effect`, and `@Environment` are valid **only** at the top
level of a component body — as a direct item of the body of a function whose
declared return type is `View`. This rule is normative and has no exceptions;
violating it is a compile-time error
([`../type-system/errors.md`](../type-system/errors.md)).

```xulo
fn helper() {
  @State let count = 0        // error: not a component body
}

fn Panel(): View {
  if true {
    @State let hidden = false // error: nested block
  }

  for i in 0...3 {
    @State let step = i       // error: nested block
  }

  VStack { }
}

async fn Load(): View {       // error: no state declarations in an `async` body
  @State let done = false
  VStack { }
}
```

- The four declarations are prohibited in plain functions, in `async`
  functions, in closures, in `if`/`for`/`while`/`match` bodies, and inside
  component blocks. Only the top level of a component body accepts them.
- **Order among themselves is free.** The declarations MAY appear anywhere
  among the top-level items of the component body: they need not be grouped at
  the front, any number of them may appear, and they MAY be interleaved with
  other statements of the body. They are statements, so the trailing expression
  of the body — the returned `View` — still comes last.
- Two state declarations with the same name in one component body are an
  error: `@State`, `@Store`, and `@Environment` each bind a name and all three
  draw from one pool per body ([`../names.md`](../names.md)). A state
  declaration MAY shadow a module-level name or a parameter, though doing so
  makes the outer name unreachable in that component.

## State and closures

A closure created while a component runs — an event handler, an effect body, a
value stored in a child — captures the **storage** of the bindings it names,
not a snapshot of their values at capture time
([`../expressions/closures.md`](../expressions/closures.md)).

```xulo
fn Counter(): View {
  @State let count: Int = 0

  let bump = fn() {
    count = count + 1
  }

  HStack {
    Button("+1", onClick: bump)
    Button("+2", onClick: fn() {
      bump()
      bump()
    })
    Text(str(count))
  }
}
```

Both handlers refer to the same cell. When either runs, it writes that cell,
and the next run of the component reads the written value: there is no copy to
fall out of date, and a handler kept across many runs keeps writing the
instance's current state.

## Examples

### Counter

```xulo
fn Counter(): View {
  @State let count: Int = 0

  VStack(spacing: 8) {
    Text(`count = ${count}`)
    Button("More", onClick: fn() { count = count + 1 })
  }
}
```

### Todo list

```xulo
struct Todo {
  title: String
  done: Boolean
}

fn TodoList(): View {
  @State let todos: List<Todo> = []
  @State let draft: String = ""

  VStack(spacing: 8) {
    Input(value: $draft, placeholder: "What needs doing?")
    Button("Add", onClick: fn() {
      todos = todos + [Todo(title: draft, done: false)]
      draft = ""
    })

    for todo in todos {
      Text(`[ ] ${todo.title}`)
    }
  }
}
```

The list and the draft are two independent cells of this instance: writing one
re-runs the component, and the run re-reads the other.

### Shared theme

```xulo
fn ThemeToggle(): View {
  @Store let { theme, setTheme } = useAppStore()

  HStack {
    if theme == Theme::Dark {
      Text("Dark mode")
    } else {
      Text("Light mode")
    }
    Button("Toggle", onClick: fn() { setTheme(Theme::Dark) })
  }
}
```

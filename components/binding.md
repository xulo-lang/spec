# Binding and Events

Component arguments pass values down from the parent that invokes a component;
event handlers run code written in that parent. Between them they cover most of
what a UI needs, but a field that the user types into also needs its value to
travel *up* into the parent's state. Xulo has two mechanisms for that: the `$`
two-way binding, which gives a child write access to a state cell of its
parent, and event handlers, which are closures that run in the parent's scope.
This chapter specifies both, and contrasts them with plain one-way arguments.

## Two-way binding `$`

`$` prefixes a name in an **argument position** of a component invocation:

```xulo
Input(value: $name)
Checkbox(checked: $isActive)
```

- `$name` passes a **binding** of the source — the variable named `name` — to
  the receiving component, rather than a copy of its value.
- The name MUST be a state variable of the enclosing component: one declared by
  `@State`, or one bound by `@Store` destructuring, in the component body that
  contains the invocation ([`state.md`](state.md)).
- A binding is not a value expression: `$name` does not evaluate to something a
  program may bind, store, index, or pass to an ordinary function. It is an
  argument form whose meaning is fixed by this section.
- What the receiving component gets is the **current value plus a setter**:
  during each run of the receiver its parameter carries the value the source
  holds at that run, and the receiver holds a channel that writes the source.
- **Writing through the setter updates the source and triggers a re-run.** The
  write lands in the same cell an assignment in the owning component would
  write, and the component that declared the source is scheduled to re-run —
  exactly as if that component had executed the assignment itself
  ([`state.md`](state.md)). When the source is a `@Store` binding, the write
  goes through the store instead, and every component that reads that store is
  re-run ([`state.md`](state.md)).

The binding is therefore a live connection, not a snapshot: the receiver reads
the source's value on each of its runs, and every write the receiver performs
comes back to the source.

## What MAY be bound

Exactly one class of names may be bound:

- **MAY** be bound: a name declared by `@State` or by `@Store` destructuring in
  the component body that encloses the invocation.
- **MUST NOT** be bound: any other name — a parameter of the enclosing
  component, an ordinary `let`, `let mut`, or `const` binding, a loop variable,
  a nested `fn`, a name from another module, or a field such as `user.name`.
  A `$` on any of these is a **compile-time error**. There is no inert binding:
  a `$` that cannot carry a write-back is rejected at compile time rather than
  accepted with writes that do nothing.

`$` is also restricted by position: it is legal only in an argument position of
a **component** invocation. Writing it in a child block, in an ordinary
expression, in a binding initializer, or in an argument to a function that is
not a component is a compile-time error.

```xulo
fn Settings(): View {
  @State let name: string = ""
  let mode = "dark"                   // ordinary binding

  VStack {
    Input(value: $name)               // OK: `name` is @State
    Input(value: $mode)               // error: `mode` is not state
    $name                             // error: `$` outside an argument
  }
}
```

## Reading the bound value

Inside the receiving component the parameter carries the current value; the
receiver does no more with it than any other argument.

```xulo
fn NameField(): View {
  @State let name: string = ""

  VStack(spacing: 8) {
    Input(value: $name, placeholder: "Your name")
    Text(`Hello, ${name}`)
  }
}
```

The field shows `name`'s current value; each edit writes the cell and schedules
a re-run of `NameField`; the re-run reads the new value, and both the input and
the greeting reflect it. A component that receives a binding MAY pass it on
(see *Binding composition* below); what its own parameter holds is the value.

## One-way property passing

The ordinary argument is one-way, and that is the default for every attribute:

| | Plain argument `value: name` | Binding argument `value: $name` |
|--------------------------|------------------------------|----------------------------------|
| What travels down | the value of `name` at that run | the current value plus a setter |
| What the receiver holds | a snapshot for that run | the source's value on each run |
| A write by the receiver | affects only the receiver's own storage | writes the source's cell |
| Owner re-runs | never, as a result of the argument | on every write through the binding |
| What the parent may pass | any expression of the parameter's type | only `@State` / `@Store` names |

A parent that wants one-way data flow passes a value; a parent that wants the
child to edit its state passes a binding.

## Event handlers

An event handler is an ordinary argument whose declared type is a function
type, written with a closure:

```xulo
fn Counter(): View {
  @State let count: int = 0

  HStack(spacing: 4) {
    Button("-", onClick: fn() { count = count - 1 })
    Text(str(count))
    Button("+", onClick: fn() { count = count + 1 })
  }
}
```

- Handlers are named callback arguments — `onClick: fn() { … }` — and follow
  the ordinary rules for arguments of function type: the value MUST be
  assignable to the declared parameter type
  ([`../types/function-types.md`](../types/function-types.md)).
- A handler is a **closure evaluated in the scope of the component that wrote
  it**: it captures the enclosing body's bindings by reference
  ([`../expressions/closures.md`](../expressions/closures.md)), so the handler
  reads and writes the same `@State` cells the body reads and writes, and a
  handler kept across runs keeps referring to them.
- A handler MAY declare parameters when the receiving parameter's function type
  declares them — `Input(onInput: fn(e) { handle(e.value) })`. With no
  annotation, the parameter's type comes from the expected function type, as
  for any closure in an argument position. Which callbacks a component
  declares, and how many parameters each has, is that component's contract.
- Handlers run **synchronously**: when an interaction resolves to a handler,
  the handler runs to completion before anything else in the program runs. If
  it assigns to `@State`, the re-run is scheduled and happens after the
  dispatch completes ([`state.md`](state.md)); an effect that follows sees the
  updated state ([`effect.md`](effect.md)).
- The language specifies that handlers are closures, when they run, and how
  they interact with state. Which components offer `onClick` or `onInput`, and
  what each one does when it is invoked, is library contract
  ([`view-syntax.md`](view-syntax.md)).

## Binding composition

A binding MAY be forwarded through an intermediate component as an ordinary
argument: the intermediate component declares parameters for the value and for
the write-back and passes them on to its children in the usual way — `$` itself
is written only where a `@State` or `@Store` name of the enclosing component is
in scope.

## Well-formedness

| Construct | Condition | Result |
|-----------|-----------|--------|
| `$name` | `name` not declared by `@State` or `@Store` in the enclosing component body | error |
| `$name` | Written outside an argument position | error |
| `$name` | Written in an argument to a function that is not a component | error |
| `$name` | Name is a field path or otherwise not a single binding | error |
| Binding argument | Parameter type not compatible with the source's type | error |
| Handler argument | Value not assignable to the declared function type | error |
| Handler argument | Parameter list incompatible with the declared function type | error |

Errors of this kind are ordinary compile-time diagnostics
([`../type-system/errors.md`](../type-system/errors.md)).

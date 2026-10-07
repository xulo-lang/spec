# View Syntax

This chapter specifies how a component is declared, how a component invocation
is written and typed, and exactly what may appear as a child inside a component
block. The declarations that give a component state, effects, and environment
are specified in [`state.md`](state.md), [`effect.md`](effect.md), and
[`environment.md`](environment.md); the measurement and painting of the
resulting tree are specified in [`layout.md`](layout.md).

## The `View` type

`View` is the declared return type of a component and the type of a fragment of
the render tree.

```xulo
fn Greeting(name: string): View {
  Text(`Hello, ${name}`)
}
```

- A value of type `View` describes one node — with its children — of the tree
  that will be laid out and painted. `View` values compose by being placed in
  the same block, not by any operation performed on `View` itself.
- `View` is **opaque to user code**. The language defines no field access, no
  method call, no subscript, and no conversion on a value of type `View`; apart
  from being returned by a component, being an element of a `list<View>`, and
  appearing as a child of another component, `View` supports nothing (see
  [`../types/primitive-types.md`](../types/primitive-types.md)).
- Componenthood is decided by exactly one thing: the declared return type. A
  function whose declared return type is `View` **is** a component, and a
  function that does not declare `View` is not. There is no additional marker,
  and a function that does not declare `View` MUST NOT use `@State`, `@Store`,
  `@Effect`, or `@Environment` — those four declarations are legal precisely at
  the top level of a component body and nowhere else ([`state.md`](state.md)).

## Declaring a component

A component is declared with `fn`, a parameter list, and the return type
`View`:

```xulo
fn Counter(): View {
  @State let count: int = 0

  VStack(spacing: 8) {
    Text(`count = ${count}`)
    Button("More", onClick: fn() { count = count + 1 })
  }
}
```

- Component parameters are ordinary typed parameters. Required, optional
  (`T?`), and default parameters, named arguments, and generic parameters all
  follow the ordinary rules ([`../functions.md`](../functions.md),
  [`../expressions/calls.md`](../expressions/calls.md)).
- The value of a component's body is its trailing expression, which MUST have
  type `View`; that value is the component's result. Statements may precede it
  in the body as in any other function body.
- A component body is not an `async` context: `await` in a component body, or
  in an `@Effect` body, is a compile-time error
  ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).
- Components are called from component blocks, from the bodies of other
  components, and from the render entry point.

```xulo
fn main(): View {
  VStack {
    Counter()
    Greeting(name: "Ada")
  }
}
```

The signature of `main` decides how a program runs:

- `fn main(): View` declares a **renderable program**. The host invokes `main`,
  takes the `View` it returns, and lays it out and paints it; every later pass
  of the program begins by invoking `main` again
  ([`../modules/source-files.md`](../modules/source-files.md),
  [`layout.md`](layout.md)).
- A plain `fn main()`, whose declared return type is omitted and is therefore
  `unit`, runs **headlessly**: the body is executed, and nothing is laid out or
  painted. A program that is never rendered still declares components, and they
  still construct `View` values — those values are simply never measured.

## Component invocations

A component invocation is a call expression: an identifier, an optional
parenthesized argument list, and an optional trailing block of children. At
least one of the argument list and the block MUST be present.

```xulo
Text("Hi")
Button("Go", variant: "primary") { Text("x") }
VStack(spacing: 8) { Text("row") }
Card { Text("no arguments") }
```

- An identifier followed by neither parentheses nor a block is **not** an
  invocation: it is a reference to the component as a value of function type,
  which MAY be passed as an argument — `Route(path: "/", component: Home)`
  passes `Home` itself.
- Arguments in `( )` follow the ordinary call rules: positional arguments come
  first and are matched in declaration order, named arguments MAY be reordered,
  and once a named argument appears every following argument MUST be named
  ([`../expressions/calls.md`](../expressions/calls.md)).
- A trailing `{ … }` block becomes the component's **children**. It is not an
  argument and is not part of the argument list. An invocation written without
  a block has no children.
- A component invocation is an expression of type `View`. It MAY be used
  wherever a `View` value is accepted: as a child in a component block, as the
  trailing expression of a component body, or as an initializer or argument of
  type `View`.
- Calling an undeclared component, calling one outside the visibility of its
  module, or supplying arguments that do not match its declared parameters is a
  compile-time error ([`../type-system/errors.md`](../type-system/errors.md)).

```text
ComponentCall   = Identifier ( '(' [ ArgumentList ] ')' [ ComponentBlock ]
                             | ComponentBlock ) ;
ComponentBlock  = '{' { ChildItem } '}' ;
ChildItem       = ComponentCall
                | 'if' Expression Block [ 'else' Block ]
                | 'for' Identifier 'in' Expression Block
                | Expression ;
```

## Children and nesting

A block belongs to the invocation it follows, and every child item belongs to
the **nearest enclosing block**: in `VStack { HStack { Text("a") } }` the
`Text` invocation is a child of the `HStack` block, not of the `VStack` block.

A component block may contain:

| Item | Form | Contributes |
|------|------|-------------|
| Component invocation | `Name(args) { … }` | one child |
| String | `"…"`, `'…'`, `` `…` `` | a text node |
| `if` / `else` | `if cond { … } else { … }` | the children of the selected branch |
| `for` | `for x in xs { … }` | one child per element |
| Expression of type `View` | `makeRow(item)` | one child |
| Expression of type `list<View>` | `rows` | one child per element, in list order |

Anything else in a component block is a compile-time error
([`../type-system/errors.md`](../type-system/errors.md)). In particular, a
binding declaration, an assignment, a `return`, or an expression of any other
type — `int`, `boolean`, `list<int>`, `map<K, V>` — MUST NOT appear as a child
item.

Component-block items are child items, not expression statements: they
contribute to the tree rather than discarding a value, so the statement-value
rule of [`../statements/README.md`](../statements/README.md) does not govern
them.

A block may be empty — `Card { }` contributes no children — and nesting depth
is unlimited: a block's items may themselves carry blocks, and the children of
each item attach to that item alone.

## Expression children

An expression written as a child item MUST have type `string`, `View`, or
`list<View>`; no other type is admissible.

- A `string` child becomes a text node. A template literal is interpolated
  before it becomes a text node, so `` VStack { `total: ${n}` } `` contributes
  one text node containing the rendered total (templates are specified in
  [`../expressions/literals.md`](../expressions/literals.md)).
- A `View` child becomes one node, whatever expression produced it: a component
  invocation, a call to a helper that returns `View`, or a parenthesized
  expression.
- A `list<View>` child **flattens**: it contributes one child per element, in
  list order, at the position of the expression — the block is as if the
  elements had been written one after another. Flattening is one level only: an
  element of a `list<View>` is itself a `View`, so a `list<View>` contributes
  exactly as many children as it has elements, at that one position.
- An expression child MAY be parenthesized — `VStack { (row) }` is
  well-formed. An expression that would begin with `{` — an object literal —
  MUST be parenthesized when written in statement-like position, exactly as
  elsewhere ([`../statements/README.md`](../statements/README.md)); an object
  literal is in any case not of an admissible child type.
- The child type is checked statically. An expression whose type is not one of
  the three admissible types is an error; in particular an optional such as
  `View?` MUST be converted to `View` first — by `??`, by `?.`, or by choosing
  a definite value with `if` or `match` (see
  [`../type-system/checking-rules.md`](../type-system/checking-rules.md)).

```xulo
fn Row(label: string, value: int): View {
  HStack {
    Text(label)
    Text(str(value))
  }
}

fn Summary(rows: list<View>, heading: View): View {
  VStack(spacing: 4) {
    "Summary"                      // string: a text node
    heading                        // View: an expression child
    rows                           // list<View>: flattened in place
    Row(label: "Total", value: 42) // component invocation: one child
  }
}
```

## Conditional and repeated rendering

`if` and `for` inside a component block choose which children are contributed.
They are the same constructs that shape ordinary computation
([`../expressions/control-flow.md`](../expressions/control-flow.md)), and they
are evaluated when the component's body runs.

```xulo
fn Banner(loggedIn: boolean): View {
  VStack {
    if loggedIn {
      Text("Welcome back!")
    } else {
      Button("Sign in", onClick: fn() { signIn() })
    }

    for item in recentItems {
      Text(item)
    }
  }
}
```

- An `if` in a block selects between child sets: the condition MUST have type
  `boolean`, exactly one branch is evaluated, and the items of that branch's
  block are contributed in place of the `if`. An `if` without an `else`
  contributes its children when the condition is `true` and **no** children
  when it is `false`.
- A `for` in a block renders one child per element. The iterable MUST be a
  `list<T>`, a `map<K, V>` (visiting its keys), a `set<T>`, or a `Range<T>`;
  the loop variable is a fresh binding for each iteration, and the body's block
  is evaluated once per element, contributing its items in order.
- Both constructs are ordinary control flow: nesting, `break`/`continue`, and
  scoping work as specified in
  [`../expressions/control-flow.md`](../expressions/control-flow.md).
- Each run of a component produces a **fresh tree** for that component: its
  blocks are evaluated again, and the children that are contributed are new
  values. What survives from run to run is state, not the tree — see
  [`state.md`](state.md) for how `@State` and `@Store` persist across runs and
  [`effect.md`](effect.md) for what runs after a run completes.

## Arguments vs children

Arguments and children answer different questions, and the language keeps them
syntactically apart:

```xulo
Button("Save", variant: "primary") {
  Text("Stores the document")
}
```

- `( )` holds **arguments**: the values the component is configured with,
  written positionally or by name, matched to the declared parameter list, and
  evaluated left to right.
- `{ }` holds **children**: the `View`s the component embeds in its own part of
  the tree, never part of the argument list and never matched to a parameter.
- **Language.** The syntax of the two positions, the argument-matching and
  evaluation rules, the typing of each child item, the flattening of
  `list<View>`, and the fact that a trailing block is children rather than an
  argument are all language rules and are specified in this chapter.
- **Library.** Which components exist, what their parameters are named and
  typed, what a particular parameter means, and which attributes a component
  accepts are all matters of the component library, not of the language. The
  convention that the first positional argument is a label or content —
  `Text("Hi")`, `Button("Go")` — comes from those declared parameter lists:
  the language matches it to the first declared parameter like any other
  positional argument. The component vocabulary is outside this specification
  (see [`../README.md`](../README.md)).

## Well-formedness

The table summarizes the compile-time errors defined by this chapter.

| Construct | Condition | Result |
|-----------|-----------|--------|
| Child item | Type other than `string`, `View`, or `list<View>` | error |
| Child item | Binding, assignment, or `return` written in a block | error |
| Component block | Written after a call to a function that does not declare `View` | error |
| `@State` / `@Store` / `@Effect` / `@Environment` | Written outside the top level of a component body | error |
| Component body | Trailing expression does not have type `View` | error |
| Component invocation | Neither an argument list nor a block | error |
| Component invocation | Positional argument after a named argument | error |
| Component invocation | Named argument that matches no declared parameter | error |
| Component invocation | Argument type not assignable to the parameter type | error |
| Component invocation | Callee not declared, or not visible at the call site | error |
| `await` | Written in a component body or an `@Effect` body | error |
| Expression child | Optional value (`View?`) used without narrowing to `View` | error |
| `if` in a block | Condition not of type `boolean` | error |
| `for` in a block | Iterable not of a supported iterable type | error |

Errors of this kind are reported as ordinary compile-time diagnostics; the
diagnostic conventions are specified in
[`../type-system/errors.md`](../type-system/errors.md).

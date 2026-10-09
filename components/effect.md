# Effects

A component's run is a description of a tree, and everything a run does is
computing that description. Work that must happen *around* a run — subscribing
to something, starting a watcher, tearing the watcher down again — is declared
with `@Effect`. This chapter specifies when an effect body runs, how a
dependency list narrows that to particular runs, how a cleanup function is
registered and invoked, and in what order effects and cleanups are processed.

## `@Effect`

An effect is declared with the `@Effect` marker and a closure, at the top
level of a component body:

```xulo
fn Counter(): View {
  @State let count: Int = 0

  @Effect fn() { print("mounted") }

  VStack {
    Text(str(count))
    Button("More", onClick: fn() { count = count + 1 })
  }
}
```

- The form is `@Effect fn() { … }`: a closure written in expression position,
  with no name — an effect binds nothing, so it draws from no name pool.
- An effect runs **after the run completes**: after the component's body has
  produced the `View` for that pass and that tree has been committed. Effects
  never contribute children and never change what the run produced.
- Effects are **not** part of a run's value. A run computes a tree; the effects
  declared by the component are then processed under the rules below.
- An effect may run on **every** re-run of the component — that is the default,
  and the next section shows how to narrow it.
- An `@Effect` body is not an `async` context: `await` in an effect body is a
  compile-time error
  ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).
- An assignment to an `@State` binding inside an effect body schedules a
  further re-run of the component, exactly as an assignment anywhere else does
  ([`state.md`](state.md)).

## Dependency lists

An effect MAY be given a dependency list: a list literal of expressions,
written after a comma following the effect's closure.

```xulo
fn UserProfile(id: String): View {
  @State let profile: String = ""

  @Effect fn() { fetchProfile(id) }, [id]

  Text(profile)
}
```

- The form is `@Effect fn() { … }, [expr, …]`. The list is evaluated once per
  pass, after the run completes, in the component's scope — it therefore sees
  the values that the run produced.
- With a dependency list, the effect runs **on mount and whenever any listed
  value changes**, and it does **not** run on re-runs in which none of the
  listed values changed.
- **Mount** is the first pass of a component instance: every effect of the
  instance runs after that pass, whether or not it has a dependency list.
- A listed value has *changed* when it is no longer `==` to the value it had at
  the previous run of that effect: each listed expression is compared with
  `==` against the value recorded for it at the effect's last run, and any
  difference — that is, any comparison that yields `false` — makes the effect
  due ([`../expressions/operators.md`](../expressions/operators.md)). Entries
  are compared one by one; a change in any one of them makes the effect due.
- Without a dependency list, the effect runs **after every re-run** of the
  component, mount included.

```xulo
fn Search(query: String): View {
  @Effect fn() { print(`searching: ${query}`) }, [query]
  @Effect fn() { print("this runs on every re-run") }

  Text(query)
}
```

## Cleanup

An effect body may return a function. Doing so registers that function as the
effect's **cleanup**:

```xulo
fn UserProfile(id: String): View {
  @Effect fn() {
    setupSubscription(id)
    return fn() { cleanupSubscription() }
  }
  // …
}
```

- When an effect body evaluates to a function, that function MUST have type
  `fn(): Unit`, and it is registered as the effect's **cleanup**. Every other
  value of the body is discarded, so a body that ends in a call — or in a value
  of any other type — registers no cleanup.
- Registering **replaces** the cleanup registered by the effect's previous run:
  an effect has at most one registered cleanup at a time.
- The registered cleanup is invoked **before that effect runs again** — after
  it has been determined that the effect is due, and before its body is
  entered — and it is invoked **when the component instance unmounts**.
- Cleanup is what makes an effect reversible: the body sets up, the cleanup
  tears down, and the rules guarantee that every set up is matched by exactly
  one tear down — either immediately before the effect runs again, or at
  unmount.

In the example above, `setupSubscription` and `cleanupSubscription` are
library functions; the language contributes only the rule that the returned
closure is the effect's cleanup.

## Ordering and lifetime

Effects and their cleanups are ordered by the order of their declarations in
the component body:

- After a run completes, the effects that are due for that pass are processed
  in **declaration order**.
- Before those effects are invoked, the cleanups registered by the effects that
  are due in that pass are invoked in **reverse declaration order**; the due
  effects then run in declaration order, each replacing its cleanup as it goes.
  An effect that is not due is not invoked, and its registered cleanup is left
  in place for a later pass.
- When a component instance unmounts — it is no longer part of the tree built
  by subsequent passes — every cleanup still registered by that instance is
  invoked, in **reverse declaration order**, and no effect of that instance
  runs again.

```xulo
fn Session(): View {
  @Effect fn() {
    acquire("a")
    return fn() { release("a") }
  }                                   // first declared

  @Effect fn() {
    acquire("b")
    return fn() { release("b") }
  }                                   // second declared

  // pass 1 (mount): invoke "a", then "b"
  // a later pass:   invoke cleanups in reverse order — "b", then "a" —
  //                 then invoke "a", then "b"
  // unmount:        invoke cleanups in reverse order — "b", then "a"
  Text("session")
}
```

Effects live exactly as long as their component instance: created on mount,
possibly re-run on later passes, torn down at unmount. Nothing about an effect
survives the instance that declared it.

## Legality

`@Effect` is valid **only** at the top level of a component body — as a direct
item of the body of a function whose declared return type is `View`. Writing it
in an ordinary function, in an `async` function, in a closure, or inside a
nested block — an `if`, a `for`, a `match`, or a component block — is a
compile-time error ([`../type-system/errors.md`](../type-system/errors.md)).
This rule is normative and has no exceptions.

| Construct | Condition | Result |
|-----------|-----------|--------|
| `@Effect` | Written outside the top level of a component body | error |
| `@Effect` body | Written with `await` | error |
| Dependency list | Not a list literal | error |
| Effect body | Value is a function whose type is not `fn(): Unit` | error |

Several effects MAY be declared in one body, in any order relative to the
other top-level items; their relative order is what orders their invocation and
their cleanup ([`state.md`](state.md)).

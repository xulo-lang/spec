# Variable Bindings and Assignment

A binding associates a name with a value; assignment writes a new value into
an existing mutable place. This chapter defines the binding forms — `let`,
`let mut`, `const`, the `:=` sugar, and struct and tuple destructuring — the
rules of assignment, and how bindings are shadowed and ordered. Scoping and
name resolution are specified in [`../names.md`](../names.md).

## `let` bindings

```xulo
let count = 0
let name: String = "Alice"
let maybe: String? = null
```

- The form is `let x = expr` or `let x: T = expr`. A type annotation, when
  present, is written after the name and separated by `:`.
- A binding introduced by `let` is **immutable**: assigning to it is a
  compile-time error (see [Assignment](#assignment)).
- **Initialization is required.** Every `let` binding is initialized by an
  expression in the same statement; there is no uninitialized binding form.
- When no annotation is written, the binding's type is inferred from the
  initializer: `let count = 0` infers `Int`. When the initializer does not
  determine the type — an empty list literal — the annotation is REQUIRED
  ([`../expressions/literals.md`](../expressions/literals.md)).
- The binding is in scope from the end of its declaration to the end of the
  innermost enclosing block ([`../names.md`](../names.md)).

## `let mut`

```xulo
let mut x = 10
let mut label = "ready"
```

- `let mut` introduces a **mutable** binding. Only a `mut` binding — or a
  mutable place reached through one — may be assigned.
- A place is mutable when its base binding is `let mut`, or when it is a
  field or subscript reached through such a binding, or through a `mut`
  receiver parameter. The ownership and borrow rules behind `mut` places are
  specified in [`../memory-and-runtime.md`](../memory-and-runtime.md).

```xulo
struct Point { x: Int, y: Int }

let mut p = Point(x: 0, y: 0)
p.x = 1              // legal: field through a mut binding

let q = Point(x: 0, y: 0)
q.x = 1              // error: q is not mut
```

- Assigning to a place that is not backed by a `mut` binding — an ordinary
  `let`, a `const`, or a place read through an immutable borrow — is a
  compile-time error ([`../type-system/errors.md`](../type-system/errors.md)).
- `mut` is part of the binding form, not of the type: `let mut x = 1` binds
  `x` as `Int`.

## `let x := expr`

```xulo
let x := 0
// desugars to
let mut x = 0

x = x + 1              // legal
```

`:=` is sugar for a mutable binding. The form `let x := expr` desugars to
`let mut x = expr`; the two forms are equivalent in every respect, including
mutability, inference, and scope.

- `:=` appears only directly after `let`, never after `let mut` or `const`:
  `let mut x := 1` and `const K := 1` are not well-formed.
- Outside this position `:=` is not an operator and does not appear in the
  expression grammar.

## `const`

```xulo
const APP_NAME = "Xulo"
const MAX_COUNT = 100
const LIMIT: Int = MAX_COUNT * 2

const A = 2
const B = A * 3            // legal: arithmetic over constants
const C = B > 5            // legal: comparison over constants

fn compute(): Int { 42 }
const BAD = compute()      // error: not a constant expression
const ZERO = 1 / 0         // error: constant division by zero
```

- `const` introduces a **compile-time constant** binding. Like `let`, it is
  immutable; there is no `const mut`.
- The initializer **MUST** be a constant expression: a literal; a reference
  to another `const` binding; or arithmetic, comparison, and other
  computation over constant expressions, using the operators of
  [`../expressions/operators.md`](../expressions/operators.md). A call — to
  a user function or to an intrinsic — is **not** a constant expression, and
  neither is any construct whose value is not determined at compile time.
- The type annotation is optional when the initializer determines the type;
  it is REQUIRED when the initializer does not.
- Constant evaluation obeys the numeric error rules: constant division or
  remainder by zero, and constant out-of-range results, are compile-time
  errors ([`../type-system/checking-rules.md`](../type-system/checking-rules.md)).
- A `const` MAY be declared at module level or inside a block. In either
  position it follows the ordinary scope rules of [`../names.md`](../names.md);
  at module level it MUST be declared before use
  ([Initialization order](#initialization-order)).
- Shadowing a `const` in a nested block is ordinary shadowing; two `const`
  declarations of the same name in the *same* block are an error.

```xulo
const SCALE = 2

fn demo(): Int {
  const SCALE = 10
  SCALE * 3        // 30: the inner constant
}

SCALE * 3          // 6: the outer constant is visible again after the block
```

## Assignment

```xulo
let mut x = 10
x = x + 1

let mut xs = [1, 2, 3]
xs[0] = 9
```

- The form is `place = expr`. The place MUST be a place expression backed by
  a mutable binding or a mutable field or borrow
  ([`../expressions/README.md`](../expressions/README.md),
  [`../memory-and-runtime.md`](../memory-and-runtime.md)).
- Evaluation order is fixed: the right-hand side is evaluated first, then the
  receiver and index subexpressions of the place left-to-right, and only then
  does the write occur.

```xulo
fn getRow(): List<Int> { [0, 0, 0] }
fn getIndex(): Int { 0 }

let mut grid = getRow()
grid[getIndex()] = 1
// evaluates `1`, then `grid`, then `getIndex()`, then writes grid[0]
```

- The result of an assignment is `Unit`. Assignment is a statement: it
  produces no meaningful value.
- Assigning to a non-`mut` binding is a compile-time error
  ([`../type-system/errors.md`](../type-system/errors.md)).
- **Compound assignment does not exist.** There is no compound-assignment
  operator and no increment or decrement operator. Write the full form:

```xulo
let mut n = 0
n = n + 1
```

## Destructuring bindings

A `struct` or a tuple may be deconstructed into several bindings at once.

### Struct destructuring

```xulo
struct Person { name: String, age: Int }

let person = Person(name: "Alice", age: 30)
let { name, age } = person
print(name)        // "Alice"
print(age)         // 30
```

- The form is `let { f1, f2, … } = expr`. Each listed field name binds a new
  **immutable** binding of the same name. The language provides no rename
  form: `let { name: n } = person` is not part of the syntax.
- The initializer MUST have a `struct` type that declares every listed
  field. Destructuring a struct that is missing a listed field is a
  compile-time error
  ([`../type-system/errors.md`](../type-system/errors.md)).
- Destructuring binds one level of fields. Deeper structure is obtained with
  field access, or with `match` when the value is an enum or a list.
- **Map, list, and enum values are not deconstructed with `let`.** Map keys
  are not statically declared — read entries with the subscript or member form
  ([`../expressions/path-and-access.md`](../expressions/path-and-access.md));
  lists and enums come apart with `match`, whose patterns are specified in
  [`../expressions/control-flow.md`](../expressions/control-flow.md).
- Component state declarations use the same struct form with an `@` prefix —
  `@Store let { user, theme } = useAppStore()` — and add the component rules
  of [`../components/state.md`](../components/state.md).

### Tuple destructuring

```xulo
let p = (10, "ten")
let (n, label) = p
let (first, _) = p            // `_` discards an element
```

- The form is `let (n1, n2, …) = expr`, with **at least two** names and a
  trailing comma allowed. It never renames: each name binds the element at
  its position under that name.
- The initializer MUST have a tuple type, and the name count MUST equal the
  tuple's arity; anything else is a compile-time error (`E0219`). Each name
  binds a new **immutable** binding of the element type at its position; `_`
  binds nothing and discards the element.
- There is no `let mut (a, b)` form. To write the elements later, keep the
  tuple itself mutable — `let mut t = (1, 2)` then `t.0 = 5`.
- The forms are exclusive: an initializer is deconstructed either with
  `{ … }` (a `struct` type) or with `( … )` (a tuple type), never both.

## Rebinding and shadowing

```xulo
fn demo(): Int {
  let x = 1
  let x = 2      // error: duplicate binding in the same block
  x
}

fn demo2(): Int {
  let x = 1
  if true {
    let x = 2    // legal: shadowing in a nested block
    print(x)     // 2
  }
  x              // 1
}
```

- Two declarations of the same name in the *same* block — any mix of `let`,
  `let mut`, and `const` — are a duplicate-declaration error
  ([`../names.md`](../names.md)).
- Shadowing is allowed in a nested block: from its declaration to the end of
  the inner block the name refers to the inner binding; outside that block
  it refers to the outer one again.
- Shadowing does not affect closures that already captured the outer binding;
  a later declaration with the same name does not re-point a captured name
  ([`../expressions/closures.md`](../expressions/closures.md)).

```xulo
fn demo3(): Int {
  let x = 1
  let read = fn(): Int { x }   // captures the outer x
  let mut total = 0
  if true {
    let x = 2
    total = x                  // total is 2
  }
  total + read()               // 2 + 1: the closure still sees 1
}
```

## Initialization order

- Inside a block, a binding is in scope from the end of its declaration to
  the end of the block. Reading a binding before its declaration in the same
  block is a compile-time error; bindings are not hoisted
  ([`../names.md`](../names.md)).
- Initializers run in declaration order, so a `let` MAY use an earlier
  binding of the same block but MUST NOT use a later one:

```xulo
let a = 1
let b = a + 1     // legal: a is already in scope
let c = b + 1     // legal
```

- Forward references apply only to module-level declarations that are not
  `let`, `let mut`, or `const`: module-level `fn`, `struct`, `enum`,
  `trait`, `type`, and `impl` MAY reference each other regardless of order,
  while module-level `let`, `let mut`, and `const` MUST be declared before
  use.

## Component bindings

`@State`, `@Store`, and `@Environment` are binding declarations that carry
extra rules:

```xulo
fn Counter(): View {
  @State let count: Int = 0
  @Store let { user, theme } = useAppStore()
  @Environment let router: Router
}
```

- Each declaration introduces a binding, and each is valid only at the top
  level of a component body — a function whose declared return type is
  `View`. Writing any of them in an ordinary function, in an `async`
  function, or inside a nested block is a compile-time error.
- `@State` bindings are implicitly mutable: assignment is allowed without
  `mut`, and each assignment schedules a re-render. `@Store` destructures
  values from a shared store; `@Environment` injects a value supplied by the
  runtime. The declaration forms, mutability, and identity across renders
  are specified in [`../components/state.md`](../components/state.md).

# Names

A name is an identifier that a declaration binds to something a program can
then refer to. This chapter defines which entities are named, the conventions
recommended for choosing names, how scopes are formed, how shadowing behaves,
and the order in which a use of a name reaches a declaration. The lexical form
of identifiers — ASCII letters, digits, and `_` — is given in
[lexical-structure.md](lexical-structure.md).

## Identifiers and Named Entities

```text
Identifier ::= Letter { Letter | Digit }
```

The following entities are named, each by an identifier:

| Entity | Introduced by | Notes |
|--------|---------------|-------|
| Variables and constants | `let`, `let mut`, `const` | Bind a name to a value |
| Parameters | A `fn` or closure parameter list | Including parameters with default values |
| Functions | `fn` at module level or inside a body | Closures (`fn() { ... }`, `=>`) are anonymous |
| Types | `struct`, `enum`, `trait`, `type` | Including type aliases of any shape |
| Components | A `fn` whose declared return type is `View` | Named exactly like any other function |
| Struct fields | A field declaration inside `struct` | Accessed with `.` |
| Enum variants | A constructor inside `enum` | Named `Enum::Variant`, never `Enum.Variant` |
| Type parameters | `<T>` in a signature or a `where` clause | Visible in the signature and the body |
| Pattern bindings | A `match` arm pattern | Scoped to that arm only |
| Namespaces | `import * as ns from "m"`; built-ins `Math`, `Time`, `Task` | Members are selected with `.` |
| Argument labels | Named parameters and component attributes | The label before `:` — `spacing`, `onClick` — is part of a signature, not a separate binding |

A module is identified by its file path, so a file declares no module name of
its own; only the namespace binding created by `import * as ns` (or by a
re-export) is an ordinary name. Argument labels are not bindings: they may not
be referenced as values, and they live in no scope.

`_` names nothing; see [Underscore and Discards](#underscore-and-discards).

## Naming Conventions

These conventions are RECOMMENDED, not enforced: breaking them produces no
diagnostic. The only name shapes the language itself rejects are keywords,
reserved words, and a lone `_` used as an ordinary identifier.

| Kind | Convention | Examples |
|------|------------|----------|
| Structs, enums, traits, type aliases, components | UpperCamelCase | `User`, `Theme`, `Area`, `VStack` |
| Enum variants | UpperCamelCase | `Theme::Dark`, `Level::High` |
| Functions, variables, parameters, fields, methods | lowerCamelCase | `sumTo`, `count`, `area`, `label` |
| Constants | UPPER_SNAKE_CASE | `PI`, `APP_NAME` |
| Type parameters | A single letter, never a descriptive word | `T`, `U`, `E`, `K`, `V` (a lowercase `t` or `e` is also a single letter) |
| Argument labels and attributes | lowerCamelCase | `spacing`, `variant`, `onClick` |

Traits are named for the capability they describe, without an `I` prefix:
`Area`, not `IArea`; `Display`, not `IDisplay`. Component names follow
UpperCamelCase, which is how a reader recognizes a component at a use site;
concrete component names such as `Text` and `Button` are ordinary imported
names, not reserved words. Names SHOULD be descriptive rather than abbreviated,
and SHOULD NOT repeat their kind in the name (`userNames` for a list of names,
not `userNamesList`).

## Reserved and Contextual Names

The keyword, reserved-word, and contextual-keyword sets are defined in
[lexical-structure.md](lexical-structure.md). None of them may be declared,
so none of them can be shadowed: `let match = 1` and `let class = 1` are
errors. `State`, `Store`, `Effect`, and `Environment` are contextual only
directly after `@`; used anywhere else they are ordinary identifiers and
follow every rule on this page.

Two groups of names are built in rather than declared:

- **Built-in type names** — `String`, `Number`, `Boolean`, `List`, `Map`,
  `Set`, `Unit`, `Unknown`, `Task`, `Range`, `View`, together with `Int`,
  `Float`, and the fixed-bit numeric names — SHOULD NOT be shadowed.
- **Intrinsic names** — `print`, `println`, `str`, `Math`, `Time` (and the
  namespace `Task`) — SHOULD NOT be shadowed. Intrinsics are described in the
  [builtins chapter](builtins/README.md).

Shadowing a built-in type name means precisely this: a declaration introduces
an ordinary binding whose spelling is the built-in type name, in a scope where
that type was visible, and from that declaration to the end of the enclosing
scope the name denotes only the new binding. The built-in type is hidden: a
use of the name in type position inside that scope — an annotation, a type
argument, a bound — no longer resolves to a type and is an error. Likewise,
shadowing an intrinsic hides it: calls to the intrinsic inside that scope no
longer reach the intrinsic. Shadowing is what makes `let print = 0` legal and
`print("x")` afterwards an error — which is why both groups SHOULD be left
alone.

## Scopes

A scope is the region of a file in which a name is visible. The language has
the following scope kinds:

| Scope | Introduced by | Names visible |
|-------|---------------|---------------|
| Module (file) scope | The file itself | Every top-level declaration, plus every `import` and `pub use` |
| Function scope | A `fn` header | Parameters, and the body that follows |
| Generic-parameter scope | `<T>` or a `where` clause | The type parameter, in the signature, return type, and body |
| Block scope | Every `{ ... }` | Declarations made inside the block |
| Closure scope | A closure (`fn() { ... }`, `=>`) | Its parameters, plus everything captured at creation |
| Pattern scope | A `match` arm pattern | Bindings made by the pattern, in that arm's body only |

Declaration order decides visibility:

- **Inside a function body**, a name is in scope from the end of its
  declaration to the end of the innermost block that encloses it. Using a
  local before its declaration in the same block is an error; locals are not
  hoisted.
- **At module level**, `fn`, `struct`, `enum`, `trait`, `type`, and `impl`
  declarations are in scope throughout the whole file, whichever order they
  appear in. A function or type MAY refer to a function or type declared
  later in the file, so mutual recursion and forward references are legal.
- **At module level**, `let`, `let mut`, and `const` are in scope only from
  the end of their declaration to the end of the file. A forward reference to
  a variable or constant MUST be an error: initializers run in declaration
  order, and there is nothing to read yet.
- Parameters are in scope from the start of the function body; a type
  parameter is in scope over the whole signature and body it belongs to.

## Shadowing

A declaration in an inner scope MAY shadow a declaration of the same name in
an outer scope. From its declaration to the end of the inner scope the name
refers to the inner binding; outside the inner scope it refers to the outer
one again. Two declarations of the same name in the *same* scope are a
duplicate-declaration error — shadowing requires a nested scope.

A shadow MUST NOT cross a closure boundary in a way that changes the meaning
of a binding a closure has already captured. A closure captures the bindings
that are in scope at the point where the closure is created; a declaration
introduced afterwards, even one with the same name in an enclosing scope, does
not re-point the captured name.

```xulo
fn demo(): Int {
  let x = 1
  let read = fn(): Int { x }   // captures the outer x
  let mut total = 0
  if true {
    let x = 2                  // shadows only inside this branch
    total = x                  // total is 2
  }
  total + read()               // 2 + 1: the closure still sees 1
}
```

Inside a closure body, an ordinary inner declaration MAY shadow a captured
name for the rest of that body; this is the same rule as any other nested
scope.

Component state declarations follow the same rules with one extra
restriction: two state declarations with the same name in one component body
are an error. `@State`, `@Store`, and `@Environment` each bind a name and all
three draw from one pool per component, so a body that binds `count` with
`@State` and binds `count` again with `@Environment` MUST be rejected. A state
declaration MAY shadow a module-level name or a parameter — that is ordinary
shadowing — but it SHOULD NOT, because the outer name becomes unreachable in
that component.

## Underscore and Discards

`_` is the wildcard name. In a pattern it matches any value and introduces no
binding: nothing is captured, the matched value is dropped, and `_` cannot be
referenced afterwards, because no name was created.

```xulo
enum Shape {
  Circle(radius: Number)
  Point
}

fn area(s: Shape): Number {
  match s {
    Shape::Circle(r) => 3 * r * r
    _ => 0                  // matches anything; binds nothing
  }
}
```

`_` is therefore never a declaration: it introduces no binding, so nothing can
be assigned to it, passed on, or read back from it, and it cannot be shadowed
because there is nothing to shadow. Wherever an ordinary identifier is
required — a member name, an import entry, a declaration — `_` alone is an
error, since it is not an identifier in the sense of
[lexical-structure.md](lexical-structure.md).

## Name Resolution Order

When a name is used, it is resolved by searching the following scopes in
order; the first scope that provides the name wins:

1. Bindings of the innermost enclosing block, innermost first.
2. Parameters and type parameters of the enclosing function or closure.
3. Remaining bindings of the enclosing function, then the module's own
   top-level declarations (functions and types in any order; variables and
   constants in declaration order).
4. Names brought into the module by `import` and by `pub use` — the module's
   interface as this file sees it.
5. The prelude: the built-in type names and the intrinsics (`print`,
   `println`, `str`, `Math`, `Time`, `Task`), which are visible only when
   nothing above matches.

Namespaces are resolved before member access. In `Math.max(a, b)` the leading
`Math` is resolved as an ordinary name — an imported namespace or a built-in
one — and only then is `max` looked up as its member. Enum variants are
resolved with `::`: in `Theme::Dark`, `Theme` resolves as a type name and
`Dark` as a variant of that type. `.` is never used to name a variant, so a
field named `Dark` and a variant named `Dark` do not collide.

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
| Type parameters | `<T>` in a generic parameter list or a `where` clause | Visible in the signature and the body |
| Const parameters | `<const N: int>` in a generic parameter list | A compile-time `int` value visible in the signature and the body |
| Pattern bindings | A `match` arm pattern | Scoped to that arm's guard and body only |
| Macros | `#[macro]`, `#[macro_attr]`, or `#[macro_derive(Target)]` on a module-level `fn`; `use` bindings | The macro name is the function identifier for `#[macro]`/`#[macro_attr]`, and `Target` for `#[macro_derive]`; macro names form their own namespace |
| Namespaces | `mod` declarations, `use … as` aliases; built-ins `Math`, `Time`, `Task`, `ast` | Members are selected with `::` |
| Argument labels | A named parameter or a component attribute | The label before `:` is the parameter's own name (`step_size`), or a component attribute declared by the component's contract (`onClick`); it is part of a signature, not a separate binding |

A module's name is the identifier of the `mod` declaration that claims it —
its identity in paths, while its file decides which module it *is*. Both
kinds of namespace binding — a `mod` name, an alias a `use` creates — are
ordinary names in the declaring module's scope. Argument labels are not
bindings: they may not be referenced as values, and they live in no scope.

`_` names nothing; see [Underscore and Discards](#underscore-and-discards).

## Naming Conventions

These conventions are RECOMMENDED, not enforced: breaking them produces no
diagnostic. The only name shapes the language itself rejects are keywords,
reserved words, and a lone `_` used as an ordinary identifier.

A word boundary in a name is written `_`. The spelling lowercases every word
of the name and joins the words with a single underscore; an acronym is
spelled as its lowercase letters (`url`, `http`), and a digit stays attached
to the word it belongs to (`log2`, `atan2`). An ordinary name carries no
leading underscore, no trailing underscore, and no doubled underscore.

**Official names.** The conventions in the table are normative for every
name this specification defines itself — built-in types, protocols,
namespaces, built-in members, and intrinsic functions: a built-in function,
method, or property is always snake_case (`code_at`, `now_nanos`), a
built-in type always UpperCamelCase (`List`, `Result`). The standard
library follows the same table; its chapters are documented outside this
specification ([`README.md`](README.md), § 1.1).

| Kind | Convention | Examples |
|------|------------|----------|
| Structs, enums, traits, type aliases, components | UpperCamelCase | `User`, `Theme`, `Area`, `VStack` |
| Enum variants | UpperCamelCase | `Theme::Dark`, `Level::High` |
| Functions, variables, parameters, fields, methods | snake_case | `sum_to`, `count`, `area`, `label` |
| Macro names | Derive targets UpperCamelCase; attribute and function macros snake_case | `Serialize`, `route`, `sql` |
| Constants | UPPER_SNAKE_CASE | `PI`, `APP_NAME` |
| Type parameters | A single letter, never a descriptive word | `T`, `U`, `E`, `K`, `V` (a lowercase `t` or `e` is also a single letter) |
| Const parameters | A single uppercase letter, never a descriptive word | `N`, `M` |
| Parameters declared as component attributes | lowerCamelCase | `spacing`, `variant`, `onClick` |

A parameter declared as a **component attribute** keeps lowerCamelCase: it
is a parameter of a component's API (`onClick`, `bgImage`, `spacing`), and
the component's contract, not the language, decides that it is an attribute.
It is still written and matched as an ordinary named argument; only its
spelling follows the component-attribute row. Every other named parameter
follows the function, variable, and parameter row: a named argument writes
the parameter's own name, so `splice`'s `delete_count` parameter is passed as
`delete_count: 2`.

Traits are named for the capability they describe, without an `I` prefix:
`Area`, not `IArea`; `Clone`, not `IClone`. Component names follow
UpperCamelCase, which is how a reader recognizes a component at a use site;
concrete component names such as `Text` and `Button` are ordinary imported
names, not reserved words. Names SHOULD be descriptive rather than abbreviated,
and SHOULD NOT repeat their kind in the name (`user_names` for a list of names,
not `user_names_list`).

A parameter declared as a component attribute keeps its lowerCamelCase
spelling even beside a snake_case function or field: its name is the
call-site vocabulary of the component API (`onClick`), and the component's
contract declares it as an attribute. Every other named parameter's label is
its parameter name and therefore follows the parameter rule. Types keep
UpperCamelCase so that an identifier beginning with a capital letter denotes
a type and an identifier beginning with a lowercase letter denotes a value or
a function.

## Reserved and Contextual Names

The keyword, reserved-word, and contextual-keyword sets are defined in
[lexical-structure.md](lexical-structure.md). None of them may be declared,
so none of them can be shadowed: `let match = 1` and `let class = 1` are
errors. `State`, `Store`, `Effect`, and `Environment` are contextual only
directly after `@`; used anywhere else they are ordinary identifiers and
follow every rule on this page.

Two groups of names are built in rather than declared:

- **Primitive type keywords** — `bool`, `string`, `number`, `int`,
  `float`, `i8`…`u64`, `f32`, `f64`, `null`, `unit`, `unknown` — are
  keywords: no declaration MAY bear their spelling, and there is nothing to
  shadow.
- **Built-in (non-primitive) type names** — `List`, `Array`, `Map`, `Set`,
  `StringBuilder`, `Task`, `Range`, `View`, `Result`, `Error` — and
  **intrinsic and namespace names** — `print`, `println`, `str`, `Math`,
  `Time`, `Task` (the namespace), and `ast` — SHOULD NOT be shadowed; this is
  a recommendation, not a restriction, and an inner binding may still shadow
  any prelude name. Intrinsics are described in the
  [builtins chapter](builtins/README.md).

Shadowing a built-in type name means precisely this: a declaration
introduces an ordinary binding whose spelling is one of the non-primitive
built-in type names, in a scope where that type was visible, and from that
declaration to the end of the enclosing scope the name denotes only the new
binding. The built-in type is hidden: a use of the name in type position
inside that scope — an annotation, a type argument, a bound — no longer
resolves to a type and is an error (`E0101`). Likewise, shadowing an
intrinsic hides it: calls to the intrinsic inside that scope no longer
reach the intrinsic. Shadowing is what makes `let print = 0` legal and
`print("x")` afterwards an error — which is why both groups SHOULD be left
alone.

## Scopes

A scope is the region of a file in which a name is visible. The language has
the following scope kinds:

| Scope | Introduced by | Names visible |
|-------|---------------|---------------|
| Module scope | The module body (file or inline `mod` block) | Every top-level declaration, plus every `mod`, `use`, and `pub use` binding |
| Function scope | A `fn` header | Parameters, and the body that follows |
| Generic-parameter scope | `<T>`, `<const N: int>`, or a `where` clause | The type or const parameter, in the signature, return type, and body |
| Block scope | Every `{ ... }` | Declarations made inside the block |
| Closure scope | A closure (`fn() { ... }`, `=>`) | Its parameters, plus everything captured at creation |
| Pattern scope | A `match` arm pattern | Bindings made by the pattern, in that arm's guard and body only |
| Macro scope | A module-level macro definition attribute; a macro `use` or `pub use` binding | The macro's fully qualified name throughout its module; a short macro name throughout the importing module |

Declaration order decides visibility:

- **Inside a function body**, a name is in scope from the end of its
  declaration to the end of the innermost block that encloses it. Using a
  local before its declaration in the same block is an error (`E0101`); locals are not
  hoisted.
- **At module level**, `fn`, `struct`, `enum`, `trait`, `type`, and `impl`
  declarations are in scope throughout the whole file, whichever order they
  appear in. A function or type MAY refer to a function or type declared
  later in the file, so mutual recursion and forward references are legal.
- **At module level**, `let`, `let mut`, and `const` are in scope only from
  the end of their declaration to the end of the file. A forward reference to
  a variable or constant MUST be an error (`E0101`): initializers run in declaration
  order, and there is nothing to read yet.
- **At module level**, a macro definition attribute is registered before any
  expansion, so its macro name is visible throughout the whole file,
  whichever order it appears in. Macro names are a separate namespace: they
  do not collide with value or type names, and two declarations with one
  macro name in one module are `E1010` rather than shadowing.
- Parameters are in scope from the start of the function body; a type
  parameter is in scope over the whole signature and body it belongs to.

A macro name resolves in the macro namespace — the module's own macro
definitions, then macro `use` bindings — and only at a macro position
(`#[derive(T)]`, `#[name(…)]`, `name!(…)`). There are no local macro bindings. In
a value position the macro name is not a value and is `E0101`; the
implementation function's own identifier is the value. For `#[macro]` and
`#[macro_attr]` the function identifier is both the value name and the macro
name; for `#[macro_derive(Target)]` the implementation function identifier is
independent of `Target`.

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
fn demo(): int {
  let x = 1
  let read = fn(): int { x }   // captures the outer x
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
are an error (`E0103`). `@State`, `@Store`, and `@Environment` each bind a name and all
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
  Circle(radius: number)
  Point
}

fn area(s: Shape): number {
  match s {
    Shape::Circle(r) => 3 * r * r
    _ => 0                  // matches anything; binds nothing
  }
}
```

`_` is therefore never a declaration: it introduces no binding, so nothing can
be assigned to it, passed on, or read back from it, and it cannot be shadowed
because there is nothing to shadow. Wherever an ordinary identifier is
required — a member name, a use entry, a declaration — `_` alone is an
error, since it is not an identifier in the sense of
[lexical-structure.md](lexical-structure.md).

## Name Resolution Order

When a name is used, it is resolved by searching the following scopes in
order; the first scope that provides the name wins:

1. Bindings of the innermost enclosing block, innermost first.
2. Parameters, type parameters, and const parameters of the enclosing
   function or closure.
3. Remaining bindings of the enclosing function, then the module's own
   top-level declarations (functions and types in any order; variables and
   constants in declaration order).
4. Names brought into the module by `mod`, `use`, and `pub use` — the
   module's structure and its view of the world as this file sees it.
5. The prelude: the built-in type names and the intrinsics (`print`,
   `println`, `str`, `Math`, `Time`, `Task`, `ast`), which are visible only
   when nothing above matches.

Qualified names are resolved before their members. In `Math::max(a, b)` the
leading `Math` is resolved as an ordinary name — a `use`-bound namespace or
a built-in one — and only then is `max` looked up as its member; the same
holds for every path. The first segment of a path resolves as a dependency
of this package (through the manifest mapping), then as a root module of
this package — a spelling that would be both is `E0104`
([`modules/import-export.md`](modules/import-export.md)). Later
segments resolve inside what the previous segment reached. Enum variants are
resolved with `::`: in `Theme::Dark`, `Theme` resolves as a type name and
`Dark` as a variant of that type. `.` is never used to name a variant, so a
field named `Dark` and a variant named `Dark` do not collide.

`ast` is a namespace name like `Math`: it is written only as the head of a
`::` path, and it is not a value. As the bullet above says, shadowing it is
permitted but discouraged. Shadowing `ast` hides every `ast::` type and
`ast::quote!` in that scope; a module-level declaration named `ast` is
`E0106`
([builtins/prelude.md](builtins/prelude.md)).

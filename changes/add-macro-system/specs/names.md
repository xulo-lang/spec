# Names — Delta: Macro Namespace

> **Delta** — adds macro names to the named-entities table, a naming row for
> macros, `ast` to the built-in names and the resolution order, and a “Macro
> scope” paragraph to the Scopes section of `names.md`. The remaining name
> rules are unchanged.

## Identifiers and Named Entities

Add this row to the entity table:

| Macros | `#Macro(…)` on a module-level `fn`; `use` bindings | The macro name is the derive `target` or the attribute/function `name`, not the implementation function's identifier; macro names form their own namespace |

The namespaces row becomes:

| Namespaces | `mod` declarations, `use … as` aliases; built-ins `Math`, `Time`, `Task`, `ast` | Members are selected with `::` |

## Naming Conventions

Add this row to the convention table:

| Macro names | Derive targets UpperCamelCase; attribute and function macros snake_case | `Serialize`, `route`, `sql` |

The row is RECOMMENDED, not enforced: a macro name comes from a string in
`#Macro` or from a derive target identifier, and the implementation function
follows the ordinary function convention. The one enforced rule is lexical: a
macro name MUST be a legal identifier and MUST NOT be a keyword or reserved
word.

## Scopes

Add this row to the scope table:

| Scope | Introduced by | Names visible |
|-------|---------------|---------------|
| Macro scope | A module-level `#Macro` declaration; a macro `use` or `pub use` binding | The macro's fully qualified name throughout its module; a short macro name throughout the importing module |

Add this bullet to the declaration-order list:

> - **At module level**, a `#Macro` declaration is registered before any
>   expansion, so its macro name is visible throughout the whole file,
>   whichever order it appears in. Macro names are a separate namespace: they
>   do not collide with value or type names, and two declarations with one
>   macro name in one module are `E1010` rather than shadowing.

A macro name resolves in the macro namespace — the module's own `#Macro`
declarations, then macro `use` bindings — and only at a macro position
(`#Derive(T)`, `#name(…)`, `name!(…)`). There are no local macro bindings. In
a value position the macro name is not a value and is `E0101`; the
implementation function's own identifier is the value.

## Built-in names

Replace the intrinsic-names clause of the “Built-in (non-primitive) type
names” bullet with:

- **Built-in (non-primitive) type names** — `List`, `Array`, `Map`, `Set`,
  `Task`, `Range`, `View`, `Result`, `Error` — and **intrinsic and namespace
  names** — `print`, `println`, `str`, `Math`, `Time`, `Task` (the namespace),
  and `ast` — SHOULD NOT be shadowed; this is a recommendation, not a
  restriction, and an inner binding may still shadow any prelude name.
  Intrinsics are described in the [builtins chapter](builtins/README.md).

Add `ast` to the prelude step of the resolution order:

> 5. The prelude: the built-in type names and the intrinsics (`print`,
>    `println`, `str`, `Math`, `Time`, `Task`, `ast`), which are visible only
>    when nothing above matches.

`ast` is a namespace name like `Math`: it is written only as the head of a
`::` path, and it is not a value. As the bullet above says, shadowing it is
permitted but discouraged. Shadowing `ast` hides every `ast::` type and
`ast::quote!` in that scope; a module-level declaration named `ast` is
`E0106`
([builtins/prelude.md](builtins/prelude.md)).

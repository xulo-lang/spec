# Names — Delta: Modules as Bindings

> **Delta** — replaces the named-entity table and the paragraph after it in
> **Identifiers and Named Entities**, the module-scope row of the **Scopes**
> table, the `use entry` sentence in **Underscore and Discards**, item 4
> of **Name Resolution Order**, and its closing resolution paragraph in
> `names.md`.

The named-entity table's namespace row and the paragraph after the table:

| Entity | Introduced by | Notes |
|--------|---------------|-------|
| Namespaces | `mod` declarations, `use … as` aliases; built-ins `Math`, `Time`, `Task` | Members are selected with `::` |

A module's name is the identifier of the `mod` declaration that claims it —
its identity in paths, while its file decides which module it *is*. Both
kind of namespace binding — a `mod` name, an alias a `use` creates — are
ordinary names in the declaring module's scope. Argument labels are not
bindings: they may not be referenced as values, and they live in no scope.

The module-scope row of the Scopes table:

| Scope | Introduced by | Names visible |
|-------|---------------|---------------|
| Module scope | The module body (file or inline `mod` block) | Every top-level declaration, plus every `mod`, `use`, and `pub use` binding |

In **Underscore and Discards**, the required-name sentence:

> required — a member name, a use entry, a declaration — `_` alone is an
> error, since it is not an identifier in
> [lexical-structure.md](lexical-structure.md).

Item 4 of **Name Resolution Order**, and the paragraph that closes the
section:

> 4. Names brought into the module by `mod`, `use`, and `pub use` — the
>    module's structure and its view of the world as this file sees it.
> 5. The prelude: the built-in type names and the intrinsics (`print`,
>    `println`, `str`, `Math`, `Time`, `Task`), which are visible only when
>    nothing above matches.

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

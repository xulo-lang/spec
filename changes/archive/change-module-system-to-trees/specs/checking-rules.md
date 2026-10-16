# Checking Rules — Delta: Duplicates, Visibility, and Paths

> **Delta** — replaces the **Duplicates and visibility** bullet of the
> checking-rules summary in `type-system/checking-rules.md`.

- **Duplicates and visibility.** Two declarations of one name in one scope
  are `E0103`, and a collision involving a `mod` or `use` binding — two
  `use` entries, a `use` against a declaration, a glob overlap, a manifest
  head against a local name — is `E0104`. A `pub` signature MUST NOT
  mention a type outside the declaring module's interface (`E0604`); every
  path a header writes must resolve to exactly one target — a member of one
  module (`E0601`, `E0602`), a package, a `mod` file, or `super` within
  bounds (`E0607`), a file at most one module declaration (`E0610`) — and
  header forms (`mod`, `use`, `pub use`) must precede the body's
  declarations (`E0605`). A path head is resolved by the manifest mapping
  once for the package, never per asking file, so a program's names do not
  depend on which module is checked first.

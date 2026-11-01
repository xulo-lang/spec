# Prelude — Delta: String Form — Display, Debug, and Format

> **Delta** — rewrites the chapter's opening sentence, adds a `Format` row to
> the built-in types table, replaces the "Built-in protocols" section, and
> replaces two rows of the "Reserved names" table of
> `builtins/prelude.md`.

The chapter's first paragraph becomes:

The prelude is what a module knows before it imports anything: the built-in
types, the built-in protocols, and the built-in namespaces. Nothing declares
the prelude, no import introduces it, and no import can take it away; it is
the constant background of name resolution
([`../names.md`](../names.md)). This chapter lists what the prelude contains
and the names it reserves. The built-in types' members are specified with the
types; the compiler-provided functions and the namespace members are specified
in [`intrinsic-functions.md`](intrinsic-functions.md).

In the built-in types table, after the `Error` row:

| `Format` | the formatting context a `display` receives | [`../types/traits.md`](../types/traits.md) |

The whole "Built-in protocols" section becomes:

### Built-in protocols

| Protocol | Method | Required by |
|----------|--------|-------------|
| `Display` | `fn display(self, format: Format): string` | interpolation in a template literal, the intrinsic `str`, the `str` domain of `List.join`, and `print` |
| `Debug` | `fn debug(self, format: Format): string` | the field rule of `#Derive(Debug)`, and `print` |

- A type implements `Display` with `impl Display for T`, whose `display`
  returns the string form of the receiver at the `Format` it is given
  ([`../types/traits.md`](../types/traits.md)).
- The base types — `int`, `float`, `bool`, `string` — convert to text
  without an implementation; every other type needs one.
- A value interpolated by `${…}` MUST be a base type or implement
  `Display` ([`../expressions/literals.md`](../expressions/literals.md));
  the same requirement is stated for `str` in
  [`intrinsic-functions.md`](intrinsic-functions.md#conversion).
- `toString` is the derived **action**, never written: every type that
  implements `Display` has `value.toString()`, defined as `display` at the
  default `Format` — `Format(precision: null, alternate: false)` — and no
  `impl` declares the method.
- A type implements `Debug` with `impl Debug for T`, or by `#Derive(Debug)`:
  `debug` returns the value's structural form, the counterpart of
  `Display`'s user-facing one. Nothing requires `Debug` for string form;
  the derive's field rule and `print` are its consumers.

In the "Reserved names" table, the value/marker-types row and the protocols
row become:

| Value and marker types | `List`, `Array`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error`, `Format` |
| Protocols | `Display`, `Debug`, `ToString` |

`ToString` stays reserved although it resolves to nothing as a trait: the
`toString` method belongs to the language, and a module-scope `trait
ToString` would take the name while no rule uses it. Written where a trait
name is expected — `impl ToString for …` — it is an unresolved name
(`E0101`).

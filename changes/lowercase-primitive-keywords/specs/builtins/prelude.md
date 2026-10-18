# Prelude — Delta: Lowercase Primitive Keywords

> **Delta** — edits the **Reserved names** table of `builtins/prelude.md`.

The table becomes:

| Category | Names |
|----------|-------|
| Value and marker types | `List`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error` |
| Protocols | `ToString` |
| Namespaces | `Math`, `Time`, `Task` |
| Intrinsic functions | `print`, `println`, `str`, `assert` |

The *Base and numeric types* row is gone — those spellings are keywords now,
stopped at the lexical layer — and `Null`, `Unit`, `Unknown` leave the
*Value and marker types* row for the same reason. The sentence after the
table stays accurate as written ("The prelude operations defined in the
next section are reserved under the same rule…").

(The rest of the chapter — the `ToString` bullet, the `str` section, the
container tables and prose — is rename-through per
[`../../tasks.md`](../../tasks.md): `Int`→`int`, `String`→`string`,
`Boolean`→`bool`, `Float`→`float` in every code position.)

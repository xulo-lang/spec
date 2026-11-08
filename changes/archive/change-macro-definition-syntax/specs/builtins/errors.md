# Macro Errors — Delta: Macro Definition Attributes

> **Delta** — replaces the `E1010` primary-span row, the `E1010` example, and
> the `E1001` quota note in `builtins/errors.md`. The other diagnostic formats
> are unchanged.

## Primary-span table

Replace the `E1010` row with:

| `E1010` | the macro definition attribute or the offending target/argument | the macro author |

## Formats

Replace the `E1001` quota note with:

```text
   = note: raise the limit in the package configuration or in
           #[macro_derive(Model, max_steps = …)]
```

Replace the `E1010` example with:

```text
error[E1010] semantic: invalid macro definition: `#[macro_derive]` requires a target
  --> macros/model.xulo:1:1
   |
 1 | #[macro_derive]
   | ^^^^^^^^^^^^^^^
   |
   = note: write `#[macro_derive(Model)]`
```

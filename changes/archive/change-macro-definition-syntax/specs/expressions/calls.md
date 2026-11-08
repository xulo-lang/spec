# Calls — Delta: Macro Definition Attributes

> **Delta** — replaces the quasi-quote example in `expressions/calls.md`. The
> macro-call and quasi-quote rules are unchanged.

## Quasi-quote

Replace the example with:

```xulo
#[macro_derive(Serialize)]
fn derive_serialize(
  input: ast::DeriveInput
): Result<ast::DeriveOutput, ast::MacroFailure> {
  let name = input.target.name
  return Result::Ok(ast::DeriveOutput {
    items: ast::quote! {
      impl Serialize for ${name} {
        fn serialize(self): Json { Json { entries: [] } }
      }
    }
  })
}
```

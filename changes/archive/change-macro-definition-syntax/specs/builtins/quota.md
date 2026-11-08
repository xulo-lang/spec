# Macro Quota — Delta: Macro Definition Attributes

> **Delta** — replaces the “Macro override” example and the quota note in
> `builtins/quota.md`. The quota values, scope, counting, and determinism
> rules are unchanged.

## Three layers

Replace the macro-override item with:

3. **Macro override.** A macro definition attribute MAY set any of
   `max_steps`, `max_memory`, and `max_depth` as named arguments:

   ```xulo
   #[macro_derive(Model, max_steps = 5000000)]
   fn derive_model(input: ast::DeriveInput): Result<…> { … }
   ```

## Failure

Replace the quota note in the example with:

```text
   = note: raise the limit in the package configuration or in
           #[macro_derive(Model, max_steps = …)]
```

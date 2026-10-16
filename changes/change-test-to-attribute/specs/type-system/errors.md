# Errors — Delta: `#Test` Wording

> **Delta** — re-words the `E0107` and `E0501` rows in
> `type-system/errors.md`. Message shapes are unchanged; only the spellings
> in triggers and examples change. The `E0108`–`E0112` rows themselves are
> introduced by
> [add-declaration-attributes](../../../add-declaration-attributes/specs/errors.md);
> the `Test` instances of `E0109`/`E0110` ride on the registry entry in
> [traits.md](../types/traits.md).

## Declaration errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0107` | a reference to the name of a `#Test` function — a call, a value use, a path, or any other mention | `helper_check()` where `#Test fn helper_check` exists | `` `helper_check` is a test function and cannot be referenced `` | [testing.md](../testing.md) |

## Type and expression errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0501` | `await` is written outside an `async` context: in a non-`async` function or closure body, in a component body, in an `@Effect` body, at module top level, or in a `#Test` function that is not marked `async` | `await t` in a plain `fn` | `` `await` outside of an `async` context `` | [async-expressions.md](../expressions/async-expressions.md) |

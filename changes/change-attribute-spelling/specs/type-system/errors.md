# Type Errors and Diagnostics — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the `E0107`–`E0112` rows of the name and declaration
> errors table, the `#[derive]` paragraph, the `E0212` row, and the `E0501`
> row in `type-system/errors.md` with the lowercase bracket spelling. No error code or
> trigger changes.

## Name and declaration errors

Replace these rows:

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0107` | a reference to the name of a `#[test]` function — a call, a value use, a path, or any other mention | `helper_check()` where `#[test] fn helper_check` exists | `` `helper_check` is a test function and cannot be referenced `` | [testing.md](../testing.md) |
| `E0108` | an attribute name that is in no registry | `#[inline]` | `` attribute `inline` is not defined `` | [traits.md](../types/traits.md) |
| `E0109` | a registered attribute on a declaration the registry does not allow it on | `#[derive(Debug)] fn main()` | `` attribute `derive` is not valid on a function `` | [traits.md](../types/traits.md) |
| `E0110` | a registered attribute whose arguments do not fit the registered shape | `#[derive]` | `` attribute `derive` takes a parenthesized list of trait names `` | [traits.md](../types/traits.md) |
| `E0111` | a well-shaped argument that names a reserved, unregistered derive target | `#[derive(Display)]` | `` `Display` is not a registered derive target `` | [traits.md](../types/traits.md) |
| `E0112` | `#[derive]` on a type that declares generic parameters | `#[derive(Debug)] struct Box<const N: int> { v: int }` | `` cannot derive `Debug` for a generic type `` | [traits.md](../types/traits.md) |

Replace the `#[derive]` paragraph with:

The expansion of `#[derive]` is an ordinary impl, so it reuses the codes the
impl rules own: a collision with a hand-written `impl Debug` — or a second
`#[derive(Debug)]` — is `E0805`, and a field or payload that is a `struct`
with no `Debug` impl is `E0801`, whose message shape
(`` `X` does not implement `Y` ``) it shares. Template interpolation, `str`,
and a rendering position that meets `unknown` — printed, expanded by a
derive, or interpolated — own `E0212` and give it the same message.

## Type and expression errors

Replace the `E0212` row:

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0212` | a value has no string form, in `${…}`, in `str`, at `print`, or in a `#[derive]` expansion | `` `${user}` `` without `Display` | `` `User` has no string form; implement `Display` `` | [traits.md](../types/traits.md) |

## Async errors

Replace the `E0501` row:

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0501` | `await` is written outside an `async` context: in a non-`async` function or closure body, in a component body, in an `@Effect` body, at module top level, or in a `#[test]` function that is not marked `async` | `await t` in a plain `fn` | `` `await` outside of an `async` context `` | [async-expressions.md](../expressions/async-expressions.md) |

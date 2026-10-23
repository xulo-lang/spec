# Type Errors and Diagnostics — Delta: E0113

> **Delta** — adds the `E0113` row after `E0112` in the “Type and
> expression errors” table of `type-system/errors.md`; every other row is
> unchanged.

### Type and expression errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0112` | `#Derive` on a type that declares type parameters | `#Derive(ToString) struct Box<T> { v: T }` | `` cannot derive `ToString` for a generic type `` | [traits.md](../types/traits.md) |
| `E0113` | a struct whose field types reach the declaring struct through fields, tuple elements, and alias expansion with no intervening indirection | `struct Node { node: Node }` | ``type `Node` has a cyclic field definition`` | [composite-types.md](../types/composite-types.md) |

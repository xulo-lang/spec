# Names — Delta: Const Parameters

> **Delta** — replaces the type-parameter entity row, adds a const-parameter
> convention row, replaces the generic-parameter scope row, and replaces item
> 2 of `## Name Resolution Order` in `names.md`; the rest is unchanged.

Replace the `Type parameters` entity row with:

| Type parameters | `<T>` in a generic parameter list or a `where` clause | Visible in the signature and the body |
| Const parameters | `<const N: int>` in a generic parameter list | A compile-time `int` value visible in the signature and the body |

Add after the `Type parameters` convention row:

| Const parameters | A single uppercase letter, never a descriptive word | `N`, `M` |

Replace the generic-parameter scope row with:

| Generic-parameter scope | `<T>`, `<const N: int>`, or a `where` clause | The type or const parameter, in the signature, return type, and body |

Replace item 2 of `## Name Resolution Order` with:

2. Parameters, type parameters, and const parameters of the enclosing
   function or closure.

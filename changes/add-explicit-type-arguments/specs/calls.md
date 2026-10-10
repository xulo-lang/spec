# Calls — Delta: Explicit Type Arguments

> **Delta** — replaces the closing paragraph of “Generic calls” in `expressions/calls.md`.

A call MAY also write the type arguments of a generic function explicitly,
between the callee and the argument list, and the written list takes the place
of inference for that call:

```xulo
let n: Int = first([1, 2, 3])    // T is Int, inferred
let t = first<Int>([1, 2, 3])    // T is Int, written
```

Whether a `<` after a callee begins such a list is decided by a purely
syntactic rule. It does so only when both conditions hold:

1. the material between that `<` and the matching `>` parses as a
   comma-separated list of types — the empty material counts as a zero-length
   list, so `f<>(…)` reaches the arity diagnostic instead of falling back to a
   comparison; and
2. the token immediately after the matching `>` is `(`.

If either condition fails, the `<` is the relational operator and the
expression is a comparison. The scan for the matching `>` counts angle depth
and skips over `(` `)`, `[` `]`, and `{` `}` groups. The decision MUST NOT
consult name resolution, and backtracking from the type-argument reading to
the comparison reading is allowed whenever the call parse does not complete.
The rule is scoped to an identifier in call position: a callee reached through
a subscript or any other postfix that does not end in an identifier takes no
list, and a call through a value of function type has no type parameters and
accepts none. `>>` and `>=` are single tokens, so the nested-bracket spacing
rule applies — `first<Map<string, int> >(m)` is well-formed and
`first<Map<string, int>>(m)` is not. The production is given in
[grammar.md](../grammar.md).

Arity, bounds, substitution without inference, and the interaction with an
expected type are specified in [`../types/generics.md`](../types/generics.md);
the diagnostics are `E0807` (a callee that accepts no list), `E0808` (a list
of the wrong length), and `E0801` (a written argument that satisfies no
bound). A call that writes no list is checked exactly as specified before this
change.

# Expressions — Delta: Closed Range Operator

> **Delta** — replaces the range row of the precedence table and the spread
> note in `expressions/README.md`.

## Precedence

The `range` row becomes:

| range | `..<` `..=` | non-assoc |

The note beneath the table becomes:

Prefix spread `...expr` exists only inside list and map literals; it is not
part of the precedence chain. The closed-range operator is `..=`; an element
that begins with `...` is a spread (see [`operators.md`](operators.md)).

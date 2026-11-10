# Lexical Structure — Delta: Closed Range Operator

> **Delta** — replaces the “Range and spread” row of the token inventory and
> the longest-match section in `lexical-structure.md`.

## Token inventory row

The `Range and spread` row of the token inventory becomes:

| Range and spread | `..<` `..=` `...` | Half-open and closed ranges; spread inside a literal |

## Longest Match

The longest-match table becomes:

| Source | Tokens | Reading |
|--------|--------|---------|
| `a..=b` | `a` `..=` `b` | `..=` as the closed-range operator between operands |
| `...b` | `...` `b` | `...` as a spread prefix inside a literal |
| `x?.y` | `x` `?.` `y` | Optional member access, not `?` followed by `.` |
| `[1..=5]` | `[` `1` `..=` `5` `]` | One element: a closed range, not a spread |
| `[...xs]` | `[` `...` `xs` `]` | One element: a spread of `xs` |
| `iffy` | `iffy` | One identifier; `if` is not split off |
| `isnot` | `isnot` | One identifier; `is` is not split off |
| `a..b` | `a` `.` `.` `b` | Two `.` tokens; `..` is not a token |
| `p.0.1` | `p` `.` `0` `.` `1` | Two chained positional accesses, not the float `0.1` |
| `p?.0` | `p` `?.` `0` | Optional positional access; `?` joins the following `.` |
| `a??b` | `a` `??` `b` | Nullish coalescing, not two `?` marks |
| `a? ?? b` | `a` `?` `??` `b` | Postfix propagation followed by `??` |

The paragraph that follows the table becomes:

Because `..` does not exist, `..<`, `..=`, and `...` are always taken whole, and
`?` only when not followed by `.` or `?`: `p?.y` and `a??b` each form one
token, and separating the marks, as in `a? ?? b`, is how the postfix
propagation operator is written next to `??`. The closed-range operator `..=`
and the spread prefix `...` are distinct tokens, so position does not
disambiguate them: `...` is the spread prefix and MAY appear only where a list
or map element begins, while `..=` is the infix closed-range operator.

# Design: the closed-range operator

## The decision

Closed ranges are written `a..=b`; half-open ranges stay `a..<b`. `..=` is a
single infix token at the range precedence level and is non-associative, like
`..<`. `...` remains a single token but is only the spread prefix of list and
map literals. No new diagnostic code is introduced.

This makes the range pair mirror the comparison pair the language already has:

| Endpoints | Comparison | Range |
|-----------|------------|-------|
| excludes the upper bound | `<` | `..<` |
| includes the upper bound | `<=` | `..=` |

## Alternatives considered

### Keep `a...b` — Swift's spelling (status quo)

**Cost.** `...` carries no mark, so “inclusive” is encoded as the absence of a
marker — the opposite of the `<` / `<=` reading the rest of the language
teaches. Worse, `...` must also be the spread prefix, so both the lexer and the
reader need a positional rule (“inside a literal an element beginning with
`...` is a spread, elsewhere it closes a range”). That rule is currently
duplicated across three chapters, and the ``0...=15`` typos show it is easy to
get wrong. Rejected.

### `a..b` / `a..=b` — Rust's spelling

**Cost.** This would also respell the half-open range, from `..<` to `..`,
turning `..` into a token and invalidating every existing `..<` example. It is
a larger break for no additional clarity: `..<` already reads as “strictly
below” and pairs perfectly with `..=` once the closed form is `..=`. Rejected
in favour of keeping `..<`.

### `a..<b` / `a..>b` — two relational marks

**Cost.** `..>` reads as a descending range and suggests iteration toward a
lower bound, which the language does not support (`start > end` iterates zero
times). It also breaks the symmetry with `<=`: there is no `..>=`. Rejected.

### `a..=b` and keep `...` as an alias

**Cost.** Two spellings for one operator means every consumer — parser,
formatter, documentation, and the positional disambiguation rule — must keep
handling both, and the specification would have to say which is canonical. The
point of the change is to remove the overload, not to duplicate it. Rejected.

### `a..=b` — chosen

It mirrors `<=`, removes `...` from the range grammar entirely, and leaves the
half-open spelling untouched.

## Consequences

### Lexing

`..=` joins the symbol-operator inventory. Because `..` does not exist,
`..<`, `..=`, and `...` are each taken whole by longest match; no new
ambiguity arises. The longest-match table gains `a..=b` → `a` `..=` `b`, the
`a...b` closed-range row is removed, and the paragraph explaining the
positional spread/range choice is replaced by a statement that the two are
distinct tokens.

### Grammar

`RangeExpression` and `RangePattern` become `( "..<" | "..=" )`. `Spread` is
unchanged (`"..." Expression`). The `SymbolOperator` production gains `"..="`.

### AST

`RangeExpr.op` becomes `'..=' | '..<'`; `RangePattern.endInclusive` still
records which form was written, now `..=` versus `..<`. No node shape changes.

### Typing and diagnostics

None. `Range<T>`, the requirement that both operands share one numeric type,
iteration, range patterns, and the `E0201` mixed-operand error are all stated
in terms of the operator's meaning, not its spelling. No code is added,
removed, or renumbered.

### Neighbouring text

Every example that writes a closed range is respelled, including the three
``0...=15`` typos, which become ``0..=15``. The spread sections are left alone
except where they contrast the two operators, because `...` still exists — it
is simply no longer a range operator.

## Migration

This is a source-breaking spelling change. A closed range `a...b` must be
rewritten `a..=b`. Because `...` remains valid as a spread, a compiler cannot
silently reinterpret every `...` as a spread; `a...b` in expression position
now fails to parse, which lets a mechanical rewrite find the affected sites.
The failure is the ordinary parse error for an unexpected token and does not
name `..=` as the fix. That is accepted for a spelling-only change: the
affected sites are exactly the `a...b` forms a rewrite searches for, so a new
diagnostic would add machinery without adding reach.

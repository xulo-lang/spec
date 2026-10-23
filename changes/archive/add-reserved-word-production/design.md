# Design: the ReservedWord production

## The decision

Append one production to the Keywords block of `grammar.md`:

```text
ReservedWord = ? one of the spellings in the reserved list of
                lexical-structure.md ? ;
```

A lexical condition in the chapter's `? … ?` notation, pointing at the
reserved list where it already lives. The index gains `ReservedWord` beside
`Keyword`.

## Alternatives considered

### Enumerate the reserved spellings in the production — `ReservedWord = "abstract" | "actor" | …`

**Cost.** Duplicates the ~60-word list that `lexical-structure.md` already
holds as its single source of truth. Two lists that must stay identical drift
on the next reserved-word change, and the drift would be silent: the grammar
and the lexical chapter would disagree while each looked complete. The
`Keyword` production inlines its set because the keyword set *is* a grammar
fact owned by this chapter; the reserved list is owned by
`lexical-structure.md` and referenced from prose today. Rejected.

### Drop `ReservedWord` from `Token` and keep only the prose

**Cost.** `Token` decides what a token *is*; the lexical chapter states that
reserved words are produced as their own tokens, so `Token` would stop
classifying a token class the language produces. Every other class in
`Token` — `Identifier`, `Keyword`, `Literal`, `Operator`, `Delimiter` — has a
production; reserved words would be the sole token kind named nowhere in the
EBNF. The prose paragraph would then have nothing to be prose *about*.
Rejected.

### Define the production in `lexical-structure.md` instead

**Cost.** The grammar chapter claims every nonterminal "is defined exactly
once, in the section that introduces it", and `Token` — which introduces the
need — lives in `grammar.md`. Splitting definitions across two files breaks
the chapter boundary README §3 sets (grammar authoritative for syntax) and
makes the index incomplete. Rejected.

## Consequences

### Grammar and lexing

- None on ambiguity or parsing. Token classification happens before parsing:
  a run is lexed first, then classified as keyword, reserved word, or
  identifier ([`../lexical-structure.md`](../lexical-structure.md)), so the
  new production describes existing lexer behavior and constrains no parser
  decision. Where a `ReservedWord` token can appear is decided exactly as
  today: nowhere a grammar expects `Identifier`, which is what makes
  `let class = 1` ill-formed.
- The `? … ?` notation is already the chapter's form for lexical conditions
  (`Whitespace`, `Letter`, `Comment`), so the production is in-register.

### Semantics

Unchanged. The reserved list, its exclusion of keywords, the treatment of
`State`/`Store`/`Effect`/`Environment` as contextual, and the naming rules of
[names.md](../names.md) are untouched.

### Diagnostics

No code changes: reserved-word-as-name remains a lexical-layer rejection with
no dedicated `E` code, exactly as before.

### Parity

No syntax is added — `Token` already names the class. The change exchanges
zero new reading for the restoration of a guarantee the chapter already
claims ("defined exactly once"), which is the parity gate in the trivial
direction: a checkable property becomes true again.

### Verification

The index/definition bijection (191 → 192 names) and the `Token` closure
(every alternative of `Token` has a production) are both greppable invariants
listed in [`tasks.md`](tasks.md).

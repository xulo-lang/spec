# Grammar — Delta: Reserved Word Production

> **Delta** — replaces the Keywords `text` block and the first row of the
> Production index in `grammar.md`.

## Keywords

The production block replaces the current one; the prose paragraph below it
is unchanged:

```text
Keyword = "and" | "as" | "async" | "await" | "bool" | "break" | "const"
        | "continue" | "copy" | "defer" | "else" | "enum" | "extern" | "f32"
        | "f64" | "false" | "float" | "fn" | "for" | "i8" | "i16" | "i32"
        | "i64" | "if" | "impl" | "in" | "int" | "is" | "isize" | "let"
        | "lock" | "match" | "mod" | "move" | "mut" | "not" | "number" | "null"
        | "or" | "panic" | "pub" | "return" | "self" | "shared" | "spawn"
        | "string" | "struct" | "super" | "trait" | "true" | "type" | "u8"
        | "u16" | "u32" | "u64" | "unit" | "unknown" | "use" | "usize" | "where"
        | "while" ;

ReservedWord = ? one of the spellings in the reserved list of
                lexical-structure.md ? ;
```

## Production index

The first index row replaces the current one:

| Nonterminal | Section |
|-------------|---------|
| `SourceFile`, `TokenSequence`, `Skipped`, `Whitespace`, `Comment`, `Token`, `Identifier`, `IdentStart`, `IdentPart`, `Letter`, `Digit`, `HexDigit`, `OctDigit`, `Keyword`, `ReservedWord`, `Literal`, `IntegerLiteral`, `FloatLiteral`, `DecInt`, `HexInt`, `BinInt`, `OctInt`, `BooleanLiteral`, `NullLiteral` | [Lexical structure](#lexical-structure) |

# Operators

Operators build expressions from operands. Every operator has a fixed
precedence and associativity, fixed operand-type rules, and a fixed result
type. This chapter defines them all; the precedence table below is normative
and is mirrored by [`../grammar.md`](../grammar.md).

## Precedence and Associativity

Levels are listed from loosest to tightest binding.

| Level | Operators | Assoc |
|-------|-----------|-------|
| assignment | `=` | right |
| ternary | `?:` | right |
| logical or | `or` | left |
| logical and | `and` | left |
| nullish | `??` | left |
| equality | `==` `!=` | left |
| relational | `<` `>` `<=` `>=` | non-assoc |
| range | `..<` `...` | non-assoc |
| bitwise or | `\|` | left |
| bitwise xor | `^` | left |
| bitwise and | `&` | left |
| shift | `<<` `>>` | left |
| additive | `+` `-` | left |
| multiplicative | `*` `/` `%` | left |
| power | `**` | right |
| unary | `!` `-` `~` `await` | prefix |
| postfix | `f(x)` `x[i]` `x.y` `x.0` `x?.y` `x?.0` `x?` | left |

Prefix spread `...expr` exists only inside list and map literals; it is not
part of this chain. An element that begins with `...` is a spread, otherwise
`...` is the closed-range operator.

Operators marked *non-assoc* MUST NOT be chained: `a < b < c` is not
well-formed. Rewrite it, as is usually intended, as `a < b and b < c`, or use
parentheses to state the intended grouping explicitly.

## Assignment

Assignment writes a value into a place expression.

```xulo
let mut x = 1
x = x + 1
```

The target MUST be a place expression, and that place MUST be backed by a
`mut` binding or by a mutable field or borrow (see
[`../memory-and-runtime.md`](../memory-and-runtime.md)). Assigning to anything
else is a compile-time error. The right-hand side is evaluated before the
place is written, and the result of an assignment expression is `unit`.
Assignment is right-associative, so `a = b = c` groups as `a = (b = c)`; since
each assignment yields `unit`, chaining is well-formed only where `unit` is an
admitted value.

Compound assignment does not exist. There is no `+=`, `-=`, `*=`, `/=`, or any
similar operator; write the full form:

```xulo
let mut n = 0
n = n + 1
```

## Ternary

The conditional operator selects one of two expressions.

```xulo
let grade = score >= 90 ? "A" : score >= 60 ? "B" : "C"
```

The condition MUST have type `boolean`; there is no truthiness. The two
branches MUST have a common type, which becomes the type of the expression
(see [`../type-system/checking-rules.md`](../type-system/checking-rules.md)).
The operator is right-associative, so `c ? a : d ? e : f` groups as
`c ? a : (d ? e : f)`. A ternary nested inside another ternary SHOULD be
parenthesized for clarity — the parentheses never change the meaning. A
ternary is not a control-flow construct, so in statement position it follows
the ordinary expression-statement rule and MUST have type `unit` (see
[`../statements/expression-statements.md`](../statements/expression-statements.md));
to run one of two branches as a statement, use `if`.

The ternary shares its `?` spelling with the postfix propagation operator. A
`?` that follows an expression is the ternary whenever `Expression ":"`
follows it — whenever a complete ternary parses — and postfix propagation
otherwise, so `c ? a : b` and `f()?` are each read as written
([`../error-handling.md`](../error-handling.md)).

## Logical

`and`, `or`, and the prefix `!` operate on booleans only.

```xulo
let ok = a > 1 and b < 2
let either = a == 0 or b == 0
print(!flag)
```

`and` and `or` short-circuit: `and` does not evaluate its right operand when
its left operand is `false`, and `or` does not evaluate its right operand when
its left operand is `true`. Both operands MUST have type `boolean`, and the
result is `boolean`. `!` takes a `boolean` and yields its negation.

Truthiness does not exist. No value of any other type is accepted where a
`boolean` is required, so `x and y` with `x: int` is a compile-time error;
write an explicit comparison instead, such as `x != 0 and y`.

## Nullish coalescing

`a ?? b` yields `a` when `a` is not `null`, and otherwise yields `b`.

```xulo
let name = user?.name ?? "anonymous"
```

`??` short-circuits: when `a` is not `null`, `b` is not evaluated. The left
operand MUST have an optional type `T?`. Typing:

| Left | Right | Result |
|------|-------|--------|
| `T?` | `T` | `T` |
| `T?` | `U` | `T \| U` |

`??` reacts only to `null`. It is not a default for `false`, `0`, or `""`, and
there is no logical-or-with-default form — those require an explicit ternary or
comparison. See [`../types/primitive-types.md`](../types/primitive-types.md).

## Equality

`==` tests equality and `!=` its negation; both are left-associative.

```xulo
let same = [1, 2] == [1, 2]
let none = maybe == null
```

Equality is structural for all value types: primitives (`boolean`, `string`,
`int`, `float`, fixed-bit numerics), `list`, `map`, `set`, tuples,
named `struct`s, and `enum`s — for an `enum`, the variant
and every payload value are compared; for a tuple, the arities must be equal
and every pair of elements is compared, so different arities are a
compile-time error rather than `false`. Both operands MUST have a common
type; comparing unrelated types is a compile-time error, so Xulo has no
cross-type equality. `null == null` is `true`, and comparing `null` with a
value requires that value's type to be optional. Values that carry identity —
rather than structural content — are compared by identity, as defined in
[`../memory-and-runtime.md`](../memory-and-runtime.md).

## Relational

`<`, `>`, `<=`, and `>=` compare two operands.

```xulo
let small = n < 10
let sorted = "apple" < "banana"
```

Operands MUST both be numeric or both be `string`. Numeric operands follow the
promotion rules of the arithmetic operators. Two `string`s compare
lexicographically by Unicode code point, not by locale. Relational operators
are non-associative: `a < b < c` MUST NOT parse and MUST be rewritten, as
`(a < b) and (b < c)` or an equivalent form. The result is always `boolean`.

## Ranges

`a..<b` builds a half-open range that excludes `b`; `a...b` builds a closed
range that includes `b`.

```xulo
for i in 0..<10 {
  print(i)
}
let slice = lo...hi
```

Both operators are non-associative and yield the built-in generic type
`Range<T>`. The operands MUST have the same numeric type, and `T` is that type:
`0..<n` with `n: int` is `Range<int>` and `0.0..<x` with `x: float` is
`Range<float>`. Mixed operand types are not promoted, so `0..<x` with
`x: float` is an error and MUST be written `0.0..<x`.

A `Range<T>` is the operand of `for … in` iteration and is usable as a range
pattern in `match`; both are specified in
[`control-flow.md`](control-flow.md). See
[`../types/primitive-types.md`](../types/primitive-types.md).

## Arithmetic

`+`, `-`, `*`, `/`, `%`, and `**` compute on numbers, with the prefix `-` for
negation.

```xulo
let sum = a + b
let square = n ** 2
let half = -x
```

- Integer division truncates toward zero: `-7 / 2` is `-3`.
- Division or remainder by zero is a compile-time error when the operands are
  constants, and a runtime trap otherwise.
- `%` takes the sign of the dividend: `-7 % 2` is `-1` and `7 % -2` is `1`.
- `**` is right-associative and binds tighter than `*`: `2 ** 3 ** 2` is
  `2 ** 9`, and `a * b ** c` is `a * (b ** c)`.
- Unary `-` binds tighter than any binary operator: `-a * b` is `(-a) * b`.

Numeric promotion applies to mixed operands: `int + float` yields `float`, and
any combination of `int` and `float` in one expression yields `float`. Two
operands of the same fixed-bit type yield that same fixed-bit type; other
mixtures of distinct numeric types are errors unless one side is a literal
that adapts to the other side's type. Overflow of a literal is a compile-time
error and of a computed fixed-bit value is a runtime trap. See
[`../types/primitive-types.md`](../types/primitive-types.md).

## String and list `+`

`+` also concatenates strings and lists.

```xulo
let who = "Xulo"
print("Hello, " + who + "!")
let all = head + tail
```

`string + string` yields `string`. `list<T> + list<U>` yields a list whose
element type is the common type of `T` and `U` — for equal element types, that
type. Mixing a `string` with any non-string is a compile-time error: convert
explicitly with the intrinsic `str(x)` first (see
[`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md)).
There is no implicit conversion anywhere in the language.

## Bitwise and Shift

`&`, `|`, `^`, `~`, `<<`, and `>>` operate on integers.

```xulo
let bits = flags & mask
let flipped = bits ^ mask
let wide = n << 2
let half = n >> 1
let inverted = ~n
```

Both operands of a binary bitwise operator MUST be integers — `int` or a
fixed-bit integer type — and both MUST have the same type; the result has that
same type. `~` is the prefix complement of an integer and yields the operand's
type. Shifting is defined as follows: `<<` fills from the right with `0`; `>>`
fills from the left with the sign bit for signed types and with `0` for
unsigned types; bits shifted out are discarded. The shift amount MUST be a
non-negative integer, and `x << 0` and `x >> 0` are both the identity
function.

In the precedence table, `&`, `^`, `|`, and the shift operators bind more
tightly than the relational and range operators but more loosely than the
arithmetic operators, and `<<`/`>>` bind tighter than `&`. Note that
`a < b | c` therefore groups as `a < (b | c)`.

## Optional chaining

`?.` is the optional member access operator. It is the only optional-chaining
form; an optional subscript does not exist.

```xulo
user?.name
user?.profile?.theme
session?.refresh()
```

When the operand of `?.` is `null`, the entire chain evaluates to `null`
without evaluating any remaining part of the chain — including method
arguments. When it is not `null`, the member is accessed normally. If `a` has
type `T?` and `T` has a member of type `U`, then `a?.b` has type `U?`. Because
every step of a chain propagates `null`, chains MAY be freely extended and
combined with `??`, as in `session?.profile?.name ?? "anonymous"`. See
[`path-and-access.md`](path-and-access.md) for the full rules.

## Spread

Prefix `...` expands a collection inside a list or map literal and appears
nowhere else: it MUST NOT be used in an argument list, a return, or any other
expression position.

```xulo
let all = [...head, ...tail]
let merged = { ...base, active: true }
```

In a list literal the operand MUST be a `list`; in a map literal it MUST
be a `map`. When a map spread and a later entry specify the same key, the
later occurrence wins. See [`literals.md`](literals.md).

## Unary summary table

| Operator | Operand | Result |
|----------|---------|--------|
| `!` | `boolean` | `boolean` |
| `-` | numeric (`int`, `float`, fixed-bit) | the operand's type |
| `~` | integer (`int` or fixed-bit) | the operand's type |
| `await` | `Task<T>` | `T` |

All four are prefix operators at the `unary` precedence level, below every
postfix operator: `!x.y` is `!(x.y)`. `await` is legal only inside an `async`
body and is specified in [`async-expressions.md`](async-expressions.md).

## Parenthesized expressions

`(expr)` groups a subexpression and changes only how it groups; it has the
value and type of `expr`. Parentheses are also required where an expression
must start with a construct that cannot begin an expression statement, such as
a map literal, and around a callee that is itself an expression:

```xulo
({ a: 1 })
(f)(5)
(2 + 3) * 4
```

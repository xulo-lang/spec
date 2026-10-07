# Types

Xulo is a statically typed language with local type inference: every expression has a type that the checker determines before the program runs, but inference never crosses a function boundary. Named records are nominal — every `struct` and `enum` declaration introduces a distinct type — while object types, unions, intersections, and optionals are structural combinations built from existing types. The numeric system has two layers: an everyday layer (`number`, `int`, `float`) and a formal fixed-bit layer (`i8`…`u64`, `f32`, `f64`) for exact layout and FFI. Function signatures are written explicitly at module boundaries: parameter types are always annotated, every `pub` function declares its return type, and an omitted return type means `unit`. This chapter defines every type constructor, the values that inhabit it, and the rules for combining and comparing types.

## Type Kinds

The kinds of type in Xulo form a closed set; a program cannot introduce a new kind of type, only new types of an existing kind (see [`generics.md`](generics.md) and [`../grammar.md`](../grammar.md)).

| Kind | Examples | Topic file |
|------|----------|------------|
| Base | `boolean`, `string` | [`primitive-types.md`](primitive-types.md) |
| Numeric | `number`, `int`, `float`, `i8` … `u64`, `f32`, `f64` | [`primitive-types.md`](primitive-types.md) |
| Null | `null` | [`primitive-types.md`](primitive-types.md) |
| Unit | `unit` | [`primitive-types.md`](primitive-types.md) |
| Range | `Range<T>` | [`primitive-types.md`](primitive-types.md) |
| View | `View` | [`primitive-types.md`](primitive-types.md) |
| Collection | `list<T>`, `map<K, V>`, `set<T>` | [`composite-types.md`](composite-types.md) |
| Object (structural) | `object`, `{ name: string, age: int }` | [`composite-types.md`](composite-types.md) |
| Tuple (positional) | `(int, string)`, `p.0` | [`composite-types.md`](composite-types.md) |
| Struct (nominal) | `struct User { … }` | [`composite-types.md`](composite-types.md) |
| Optional | `T?` (shorthand for `T \| null`) | [`composite-types.md`](composite-types.md) |
| Union | `T \| U` | [`composite-types.md`](composite-types.md) |
| Intersection | `T & U` | [`composite-types.md`](composite-types.md) |
| Type alias | `type ApiResponse<T> = { … }` | [`composite-types.md`](composite-types.md) |
| Function | `fn(A): R`, `async fn(A): B` | [`function-types.md`](function-types.md) |
| Task | `Task<T>` | [`function-types.md`](function-types.md) |
| Enum | `enum Theme { … }` | [`enums.md`](enums.md) |
| Trait | `trait Area { … }` | [`traits.md`](traits.md) |
| Type parameter | `T`, `<T: Area>` | [`generics.md`](generics.md) |

Assignability, subtyping, and inference constraints between these kinds are specified in [`type-relations.md`](type-relations.md).

## Type Syntax

Types are written with the following grammar (the complete grammar is [`../grammar.md`](../grammar.md), which is authoritative for syntax):

```text
Type            = UnionType ;
UnionType       = IntersectionType { "|" IntersectionType } ;
IntersectionType = PostfixType { "&" PostfixType } ;
PostfixType     = PrimaryType { "?" } ;
PrimaryType     = TypeName [ "<" TypeList ">" ]
                | "(" Type ")"
                | "{" [ FieldType { "," FieldType } [ "," ] ] "}"
                | [ "async" ] "fn" "(" [ FnParams ] ")" [ ":" Type ]
                | Literal ;
FnParams        = FnParam { "," FnParam } ;
FnParam         = [ Identifier ":" ] Type ;
TypeName        = Identifier ;        (* built-in, struct, enum, trait, or alias name *)
Literal         = StringLiteral | IntegerLiteral | FloatLiteral | BooleanLiteral ;
TypeList        = Type { "," Type } ;
```

`?` binds tighter than `&`, which binds tighter than `|`; parentheses group. The return type of a function type extends as far as possible, so `fn(): A | B` is a function returning a union, while `(fn(): A) | B` is a union containing a function type. Type syntax in context — where annotations are required, and how literals are checked against them — is specified in [`../type-system/`](../type-system/README.md).

## Inference at a Glance

- Inference is **local**: it operates inside a function body or a single expression, using annotations, literal defaults, and the surrounding context. Nothing is inferred across a function boundary.
- A binding without an annotation takes the type of its initializer: `let x = 42` infers `int`, `let s = "hi"` infers `string`, `let xs = [1, 2]` infers `list<int>`.
- Literals default by form: integer forms (including `0xff`, `0b1010`, `0o77`) default to `int`, float forms to `float` (see [`primitive-types.md`](primitive-types.md)).
- Parameter types of a declared `fn` are never inferred; they MUST be annotated. The return type is declared or omitted, in which case it is `unit`. A closure MAY omit annotations that the expected type or its body determines (see [`type-relations.md`](type-relations.md)).
- Generic type arguments are inferred at each call site; the language has no explicit type arguments at call sites (see [`generics.md`](generics.md)).
- An annotation supplies context outward: `let w: u32 = 800` checks `800` as `u32` and coerces it (see [`../type-system/coercion.md`](../type-system/coercion.md)).
- The complete rules for checking, compatibility, and subtyping are in [`type-relations.md`](type-relations.md).

## How to Read This Chapter

- [`primitive-types.md`](primitive-types.md) — `boolean`, `string`, the two-layer numeric system, `null`, `unit`, `View`, and `Range<T>`.
- [`composite-types.md`](composite-types.md) — `list`, `map`, `set`, structural object types, positional tuples, nominal `struct` records, optional/union/intersection types, and type aliases.
- [`function-types.md`](function-types.md) — the syntax and meaning of `fn(...)` types, function and closure values, default and named parameters, `async`/`Task<T>` types, subtyping, and method types.
- [`generics.md`](generics.md) — type parameters, bounds (`<T: Trait>`, `where` clauses), and call-site inference.
- [`enums.md`](enums.md) — enum declarations, variants with payloads, and `Enum::Variant` paths.
- [`traits.md`](traits.md) — trait declarations, `impl` blocks, and explicit `Trait.method` dispatch.
- [`type-relations.md`](type-relations.md) — assignability, subtyping, structural vs. nominal identity, and how inference interacts with each kind.

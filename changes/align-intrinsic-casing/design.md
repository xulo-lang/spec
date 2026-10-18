# Design: intrinsic casing

## The decision

Every multi-word prelude intrinsic and the `ToString` method are spelled in
lowerCamelCase, matching the convention table in
[`../../names.md`](../../names.md): `mapGet`, `setInsert`, `listRemove`,
`toString`. The rename is spelling-only.

## Alternatives considered

### Document a prelude exception in `names.md`

**Cost.** The single-word intrinsics already satisfy lowerCamelCase, so an
exception would cover exactly the multi-word names — an arbitrary carve-out
with no semantic content. It would also teach readers that the language's
own library is a place where the conventions do not apply, and every future
intrinsic would have to pick a side. Rejected.

### Keep snake_case

**Cost.** The status quo is the inconsistency the change removes: a
convention table beside a prelude that ignores it. Nothing else in the
language surface is snake_case. Rejected.

### Convert the operations to methods — `m.get(k)`

**Cost.** A different language-design question — method surface on built-in
types, receiver syntax, and whether `Map` grows an inherent impl — with no
bearing on casing. Out of scope; the operations stay free functions.

## Consequences

### `setSize` reads like a setter

`set_size` → `setSize` is the one new name whose camel form can be misread
as the verb "set (a) size". The family rule wins: every container operation
is `<Container><Rest>` — `mapSize`, `listSize` sit beside it, and the
parameter list `(s: Set<T>): Int` disambiguates in place. A deliberate
irregularity (`setSizeOf`?) would break the family for a reader's first
glance; accepted as-is.

### Reservation shifts with the spelling

The prelude reserves its names by reference to its declarations, not by a
frozen list ([`../../builtins/prelude.md`](../../builtins/prelude.md),
Reserved names; `E0106`). After the rename, `map_get` is an ordinary legal
module-scope name and `mapGet` is the reserved one. No wording change is
needed; the reservation rule is already "whatever the prelude declares".

### No diagnostic changes

No `E0xxx` message or trigger quotes an intrinsic or method name — `E0106`'s
example uses `print`, which is unchanged; `E0212` speaks of `ToString`
implementors generically. The errors chapter needs no delta.

### Neighbouring unaccepted change

[`changes/add-test-title`](../add-test-title/proposal.md) replaces the
testing chapter's Declaration example, which currently calls `map_get`. If
that change is still unaccepted when this one folds, its delta example is
updated to `mapGet` in the same pass (tracked in
[`tasks.md`](tasks.md)); if it folded first, the sweep below catches it.

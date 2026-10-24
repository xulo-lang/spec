# Intrinsic Functions — Delta: Time Namespace Casing

> **Delta** — replaces the `Time` table, the two clock bullets, and the
> elapsed-time example in the `Time` namespace section of
> `builtins/intrinsic-functions.md`, and the five `Time::` rows of the
> complete signature index; everything else in the file is unchanged.

## `Time` namespace

`Time` is a built-in namespace of timestamps and async sleep, available
without `use`. Its functions return scalars only: `u64` for the clocks and a task for
the delay.

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Time::now` | `fn(): u64` | current Unix time in milliseconds |
| `Time::nowNanos` | `fn(): u64` | current Unix time in nanoseconds |
| `Time::monotonic` | `fn(): u64` | current reading of the monotonic clock in milliseconds |
| `Time::monotonicNanos` | `fn(): u64` | current reading of the monotonic clock in nanoseconds |
| `Time::sleep` | `fn(ms: u64): Task<unit>` | delays for at least `ms` milliseconds |

- `Time::now` and `Time::nowNanos` read the wall clock: milliseconds and
  nanoseconds since the Unix epoch. They are appropriate for timestamps, not
  for measuring how long something took.
- `Time::monotonic` and `Time::monotonicNanos` read a clock that never runs
  backwards and is unaffected by changes to the wall clock. Only differences
  between two readings are meaningful, which makes them the right choice for
  elapsed time, debounce windows, and benchmarks.
- `Time::sleep` produces a `Task<unit>` that settles once the delay has elapsed.
  A `Time::sleep(...)` call MUST be awaited: it is written
  `await Time::sleep(ms)` inside an `async` body, where it suspends that body
  for at least `ms` milliseconds and yields `unit`. `await` is legal only
  inside an `async` body
  ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).

```xulo
let start = Time::monotonicNanos()
// ... work ...
let elapsed = Time::monotonicNanos() - start

async fn later() {
  await Time::sleep(1000)
  println("1 second later")
}
```

## Complete signature index

(unchanged except for the five `Time::` rows, which keep their
alphabetical order — `monotonic` < `monotonicNanos` < `now` <
`nowNanos` < `sleep`)

| Name | Signature | Section |
|------|-----------|---------|
| `Time::monotonic` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::monotonicNanos` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::now` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::nowNanos` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::sleep` | `fn(ms: u64): Task<unit>` | [`Time` namespace](#time-namespace) |

# Concurrency — Delta: Snake-Case Service Method

> **Delta** — replaces the `onMessage` occurrences in the “Services”
> section and the service-lifetime bullet in `concurrency.md`.

## Services (opening sentence)

A **service** is an actor: a struct with an `on_message` method, running in
its own process, reached only by sending messages.

## Services (example)

```xulo
struct Tally { total: int }

impl Tally {
  fn on_message(mut self, cmd: int) {
    self.total = self.total + cmd
    reply(self.total)
  }
}

async fn main() {
  let worker = spawn.process Tally { total: 0 }
  let sum = await worker.send(41)
  print(sum)                        // 41
  worker.stop()                     // the service ends; nothing else is queued
}
```

## Services (bullets)

- `Type` MUST declare `on_message` in an `impl` block; a service type without
  one is a compile-time error (`E0409`). The method takes a receiver (`self`
  or `mut self`) and exactly one message parameter of some type `M`; it MAY be
  declared `async`.
- A service handles one message at a time, in arrival order: `on_message` runs
  to completion — through every `await`, if it is `async` — before the next
  message begins.
- `reply(e)` may be called only inside `on_message`. The reply type `R` is the
  common type of the `reply` operands in the method (`E0211` if they have
  none), or `unit` when the method never calls `reply`. The send's task
  settles when `on_message` returns, carrying the operand of the `reply` the
  executed path performed — or, when that path performed none although the
  method contains `reply` calls, settling as cancelled.

## Service lifetime and `stop` (bullet)

- `handle.stop()` requests that the service stop. Stopping is cooperative: it
  follows the same rule as a cancelled task
  ([Cancellation](#cancellation)) — the service finishes its current
  synchronous run and stops at its next `await`. A body that never awaits
  again runs to completion, and the service has stopped when `on_message`
  returns.

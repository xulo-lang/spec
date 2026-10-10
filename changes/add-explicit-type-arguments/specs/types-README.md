# Types — Delta: Explicit Type Arguments

> **Delta** — replaces the “Generic type arguments are inferred at each call site …” bullet of “Inference at a Glance” in `types/README.md`.

- Generic type arguments are inferred at each call site; a call MAY instead
  write them explicitly, in a list between the callee and the argument list —
  `first<int>(…)`, `map<string, int>(…)` — which takes the place of inference
  for that call (see [`generics.md`](generics.md)).

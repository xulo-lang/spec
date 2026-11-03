# Tasks

## Fold the normative deltas

- [ ] Apply [`specs/names.md`](specs/names.md) to `names.md` (Naming
      Conventions).
- [ ] Apply [`specs/builtins/prelude.md`](specs/builtins/prelude.md) to
      `builtins/prelude.md` (`to_string`, `Map::from_entries`).
- [ ] Apply [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md)
      to `builtins/intrinsic-functions.md` (Integer, Formatting, and `Time`
      tables; prose; examples; signature index).
- [ ] Apply [`specs/builtins/README.md`](specs/builtins/README.md) to
      `builtins/README.md`.
- [ ] Apply [`specs/types/traits.md`](specs/types/traits.md) to
      `types/traits.md` (`to_string`).
- [ ] Apply [`specs/types/composite-types.md`](specs/types/composite-types.md)
      to `types/composite-types.md` (`List`/`Array` members, `Map`/`Set`
      static constructors).
- [ ] Apply [`specs/types/primitive-types.md`](specs/types/primitive-types.md)
      to `types/primitive-types.md` (`byte_len`, `code_at`).
- [ ] Apply [`specs/functions.md`](specs/functions.md) to `functions.md`.
- [ ] Apply [`specs/expressions/path-and-access.md`](specs/expressions/path-and-access.md)
      to `expressions/path-and-access.md`.
- [ ] Apply [`specs/type-system/checking-rules.md`](specs/type-system/checking-rules.md)
      to `type-system/checking-rules.md`.
- [ ] Apply [`specs/type-system/errors.md`](specs/type-system/errors.md) to
      `type-system/errors.md` (`E0409` message).
- [ ] Apply [`specs/concurrency.md`](specs/concurrency.md) to
      `concurrency.md` (`on_message`).
- [ ] Apply [`specs/error-handling.md`](specs/error-handling.md) and
      [`specs/memory-and-runtime.md`](specs/memory-and-runtime.md) to their
      closed failure tables.

## Sweep the example code

Every Xulo identifier in an example follows the new table. Type names, enum
variants, type and const parameters, and parameters declared as component
attributes are left alone; every other named argument's label is its
parameter's name, so it changes with the parameter.

- [ ] `names.md`: `sumTo` → `sum_to`, `userNames` → `user_names`, and the
      discouraged counterexample `userNamesList` → `user_names_list` (it
      stays the negative example, not a second recommendation).
- [ ] `introduction.md`: `sumTo` → `sum_to`.
- [ ] `functions.md`: `areaOf` → `area_of`, `perimeterOf` → `perimeter_of`,
      `doubleAsync` → `double_async`.
- [ ] `types/generics.md`, `types/traits.md`: `areaOf` → `area_of`,
      `perimeterOf` → `perimeter_of`, `areaTwice` → `area_twice`.
- [ ] `expressions/async-expressions.md`: `fetchUser` → `fetch_user`,
      `stepA`/`stepB` → `step_a`/`step_b`, `priceOf` → `price_of`,
      `sumPrices` → `sum_prices`, `readCache` → `read_cache`, `logUser` →
      `log_user`, `isCached` → `is_cached`, `findUser` → `find_user`,
      `fetchCount` → `fetch_count`, `fetchAll` → `fetch_all`, `decodeUser` →
      `decode_user`.
- [ ] `expressions/closures.md`: `makeCounter` → `make_counter`,
      `makeAdder` → `make_adder`.
- [ ] `expressions/calls.md`: `isSquare` → `is_square`, `getFn` → `get_fn`.
- [ ] `expressions/path-and-access.md`: `getBox` → `get_box`.
- [ ] `types/function-types.md`: `countAny` → `count_any`, `makeAdder` →
      `make_adder`.
- [ ] `statements/let-and-assignment.md`: `getIndex` → `get_index`,
      `useAppStore` → `use_app_store`, `getRow` → `get_row`.
- [ ] `components/view-syntax.md`: `loggedIn` → `logged_in`, `signIn` →
      `sign_in`, `recentItems` → `recent_items`, `makeRow` → `make_row`.
      Keep the declared `onClick` attribute parameter.
- [ ] `components/state.md`, `components/environment.md`,
      `components/effect.md`: `setTheme` → `set_theme`, `useAppStore` →
      `use_app_store`, `setupSubscription` → `setup_subscription`,
      `cleanupSubscription` → `cleanup_subscription`, `fetchProfile` →
      `fetch_profile`. Keep the declared `onClick` attribute parameter.
- [ ] `components/binding.md`: `isActive` → `is_active`. Keep the declared
      `onClick` and `onInput` attribute parameters.
- [ ] `components/layout.md`: keep the declared `bgImage` attribute
      parameter.
- [ ] `concurrency.md`: `totalOnline` → `total_online`, `localData` →
      `local_data`, `localConfig` → `local_config`, `fetchCount` →
      `fetch_count`, `uploadFiles` → `upload_files`, `refreshOk` →
      `refresh_ok`, `refreshBad` → `refresh_bad`, `loadConfig` →
      `load_config`.
- [ ] `ffi.md`: `cMalloc` → `c_malloc`, `cFree` → `c_free`, `asInt` →
      `as_int`, `withBuffer` → `with_buffer`, `roundTrip` → `round_trip`,
      `mustAllocate` → `must_allocate`.
- [ ] `testing.md`: `testParse` → `test_parse`, `parseOk` → `parse_ok`,
      `delayedTaskFinishes` → `delayed_task_finishes`,
      `awaitNeedsTheAsyncMarker` → `await_needs_the_async_marker`,
      `awaitWithTheAsyncMarker` → `await_with_the_async_marker`,
      `referenceIsForbidden` → `reference_is_forbidden`,
      `absentKeyIsNull` → `absent_key_is_null`.
- [ ] `error-handling.md`: `findUser` → `find_user`, `fetchName` →
      `fetch_name`, `stillWrong` → `still_wrong`, `readConfig` →
      `read_config`, `pollWithDeadline` → `poll_with_deadline`, `getJson` →
      `get_json`, `fetchUser` → `fetch_user`, `doubledHalf` → `doubled_half`,
      `doubleOrStop` → `double_or_stop`, `decodeUser` → `decode_user`.
- [ ] `type-system/errors.md`: `helperCheck` → `helper_check`, `totalOnline`
      → `total_online`, `fetchCount` → `fetch_count`.
- [ ] `type-system/coercion.md`: `someInt` → `some_int`.
- [ ] `types/composite-types.md`: `setStatus` → `set_status` in the trait
      example.
- [ ] `modules/import-export.md`, `modules/visibility.md`: `rejectsEmpty` →
      `rejects_empty`, `appName` → `app_name`.
- [ ] `builtins/intrinsic-functions.md`: `fetchAll` → `fetch_all`,
      `fetchUser` → `fetch_user`, `halfPi` → `half_pi` in examples.
- [ ] `grammar.md`: keep the declared `onClick` attribute parameter.

## Verify

- [ ] `rg '\b[a-z][a-zA-Z0-9]*[A-Z][a-zA-Z0-9]*\b'` over the chapters
      returns only type names, enum variants, parameters declared as
      component attributes, the AST schema (`ast.md`), the documentation JSON
      schema (`documentation-comments.md`), and the JavaScript snippet in
      `machine-readable.md`.
- [ ] No named argument spells `deleteCount:`; every named-argument label
      matches its parameter's declaration.
- [ ] No `E` code is added, removed, or renumbered.
- [ ] No production in `grammar.md` changes; `Identifier` already admits `_`.
- [ ] The reserved-name list in `builtins/prelude.md` still names only the
      unchanged built-in types, protocols, and namespaces; no renamed
      intrinsic needs a new entry.
- [ ] On acceptance, move this directory to `changes/archive/`, mark it
      `accepted`, and record it in `changes/README.md`.

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.loadpaths.ex
<!-- sha256: 902bfea4fc537995fd587ab607b5d325369c9ce6c580207fd7cc76120045341c -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.loadpaths.ex
<!-- sha256: 902bfea4fc537995fd587ab607b5d325369c9ce6c580207fd7cc76120045341c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`deps.loadpaths.ex` is a small, mature Mix task that checks/compiles/loads dependencies and starts pub/sub listeners; its only public entry point is `run/1`. I found no in-scope behavioural bugs. In this pass I read the three collaborator modules the first pass had left as unverified assumptions (`Mix.Dep.check_lock/1`, `Mix.Dep.load_paths/1`, `Mix.Dep.load_and_cache/0` in `lib/mix/dep.ex`, and `Mix.PubSub.start/0` + `start_listeners/0` in `lib/mix/pubsub.ex`) and confirmed the return-shape and no-raise contracts the control flow here relies on. The hedges from the first pass are resolved below.

## Bugfixes

No findings.

## Risks / caveats

- I reviewed the full file (206 lines). The only documented public API is `run/1`; all other functions are `defp` and reachable only via `run/1`'s controlled call sequence.
- The first pass declined to verify the `Mix.Dep`/`Mix.PubSub` collaborators. I have now read them (paths above). The relevant contracts are:
  - `Mix.Dep.check_lock/1` (`dep.ex:445-465`) always returns a `%Mix.Dep{}` — either with an updated `:status` or the dep unchanged — via a `case scm.lock_status(opts)` whose three clauses (`:mismatch`/`:outdated`/`:ok`) cover the documented `Mix.SCM` `lock_status` return set. It performs no I/O that raises for a well-formed loaded dep. This confirms the `Enum.map(fn {:ok, dep} -> dep end)` at line 129 cannot hit an unmatched clause under contract.
  - `Mix.Dep.load_paths/1` (`dep.ex:520-523`) always returns a one-element list, so the `Enum.flat_map(all, &Mix.Dep.load_paths/1)` at line 76 cannot produce a non-list element.
  - `Mix.PubSub.start/0` and `start_listeners/0` (`pubsub.ex:22-75`) are idempotent (guarded by `Process.whereis`), return `:ok`, and `start_listeners/0` calls `Subscriber.flush()` exactly once on first successful start — matching the accumulate-then-flush contract the header comment at lines 38-50 describes.

### Ruled out

- **`Task.async_stream(&check_lock/1, …) |> Enum.map(fn {:ok, dep} -> dep end)` (lines 128–129) crashing on a non-`{:ok, _}` result.** Verified against `Mix.Dep.check_lock/1` (`dep.ex:445-465`): for a valid loaded dep it returns a `%Mix.Dep{}` and does not raise, and with `timeout: :infinity` no `{:exit, :timeout}` is emitted. The only way to reach the unmatched clause is `check_lock/1` raising/exiting, which is outside its contract; propagating such a crash is the intended behaviour, not a defect. Confidence: high.
- **Listeners never started when `--no-deps-check` is passed without `--listeners` (lines 81–84).** Confirmed intended per the header comment (lines 38-50) and the `Mix.PubSub` module doc (`pubsub.ex:16-20`): the subscriber (`Mix.PubSub.start/0`) accumulates events and only replays them on `start_listeners/0`, precisely so listeners (which come from deps) don't run before deps are known-compiled. Accumulation is bounded by the single Mix invocation's process lifetime. Not a leak. Confidence: high.
- **`--no-optional-deps` filtering `all` (lines 65–70) before both `deps_check` and `Code.prepend_paths`.** Filtering optional deps out of every downstream consumer is the documented meaning of the option; consistent use across check and load-path is correct. Confidence: high.
- **`--no-compile` yielding `:ok` while paths are still prepended (line 76 vs. line 137).** Prepending available load paths regardless of compilation matches the documented `--no-deps-check`/`--no-compile` semantics ("only load available ones"). Confidence: high.
- **`check_elixir_version/1` `case` on `Version.parse_requirement/1` (lines 91–102).** Handles both documented return shapes (`{:ok, req}` and `:error`) and raises a specific, actionable `Mix.ElixirVersionError` / `Mix.raise`; no missing clause. Confidence: high.
- **`--listeners` forcing `start_listeners/0` even alongside `--no-deps-check` (line 81).** `--listeners` is an undocumented internal override; when present, listeners start regardless of the deps-check gate. `start_listeners/0` is idempotent and returns `:ok`; if listener modules from uncompiled deps were unavailable the failure would surface from the supervisor start, not from this file. This is pre-existing intended override behaviour, not a defect introduced here. Confidence: medium. *(Not machine-verified: exercising the both-flags path requires a full Mix project fixture with a configured listener module; assessed by reading `Mix.PubSub.start_listeners/0`.)*
- **Comment typos ("sterning", "adding adding", "compite", "loadspaths").** Cosmetic only. Confidence: high.

## Did I verify this?

- I did not execute `run/1` end-to-end; doing so requires a full Mix project fixture with real dependencies, an SCM, and a build lock. Instead of leaving the collaborator contracts as first-pass assumptions, I read them directly:
  - `lib/mix/dep.ex` — `check_lock/1` (445-465), `load_and_cache/0` (113-124), `load_paths/1` (520-523), `ok?/1`/`available?/1` (470-479). These are pure/contract-bounded and confirm the return shapes `run/1` and `deps_check/2` depend on.
  - `lib/mix/pubsub.ex` — `start/0` (22-39) and `start_listeners/0` (56-75), confirming idempotency, `:ok` returns, and the single `flush()` on first start.
- The one remaining medium-confidence item (the `--listeners` + `--no-deps-check` override) is not machine-verified because it needs a listener-configured fixture; it is dismissed on the read of `start_listeners/0` plus the option's override semantics, and flagged as such above rather than asserted as fully verified.

### Dropped in re-review

No prior findings were dropped. The first pass reported zero bugfix findings; all ruled-out entries are preserved, with the two first-pass hedges ("collaborators outside this file", "reasoned, not machine-verified") resolved by reading `lib/mix/dep.ex` and `lib/mix/pubsub.ex` as described above.

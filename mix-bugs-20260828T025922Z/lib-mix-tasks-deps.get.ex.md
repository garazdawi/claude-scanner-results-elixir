# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.get.ex
<!-- sha256: c96ea90c9782a3539d8747de125e4479278a6115ef03f0b0d0fe01f50452395d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:50:03Z_

## Summary

`Mix.Tasks.Deps.Get.run/1` is a thin, single-clause task that parses three switches, converts them to atom-keyed fetch options, and delegates to `Mix.Dep.Fetcher.all/3` under the deps lock; the logic is small and I found no genuine reachable defect in it.

## Bugfixes

No findings.

I inventoried every candidate class against the file and verified the load-bearing behaviors by execution:

- **Option→atom conversion (line 38-41).** The comprehension filter `value = opts[switch]` correctly drops absent (`nil`) and `false` switches, and `String.to_unsafe_atom("#{value}")` produces the intended terms for all three switches. Verified: `[only: "prod", target: "host", check_locked: true]` → `[env: :prod, target: :host, check_locked: true]`, and `Keyword.get(fetch_opts, :check_locked, false)` → `true`. The `check_locked` boolean survives the string round-trip because the atom `:true` is the boolean `true` (same term), so `Mix.Dep.Lock.write/2`'s `Keyword.get(opts, :check_locked, false)` guard fires correctly.
- **Downstream key contract.** `env`/`target` must be atoms because `Mix.Dep.Converger` builds `{opts[:env], opts[:target]}` and compares against dep `:only`/`:targets` atom lists — so the atom conversion is required and correct, not gratuitous.
- **`Mix.Dep.Fetcher.all(%{}, Mix.Dep.Lock.read(), fetch_opts)` (line 43).** Argument order matches `all(old_lock, new_lock, opts)`; empty `old_lock` with the current lock as `new_lock` is the correct "consider everything changed" shape for `deps.get`.
- **Success message (line 46).** `"All dependencies have been fetched"` matches the test assertion in `deps.git_test.exs:244`.
- **`with_deps_lock/2` call (line 32).** The single-arg call binds the closure to `fun` and uses the default `config()` — valid.
- **`String.to_unsafe_atom/1` existence.** Confirmed present in this build (and already used at `mix/dep/fetcher.ex:164`), so no `UndefinedFunctionError`.

## Risks / caveats

- **Atom creation from CLI values.** `String.to_unsafe_atom("#{value}")` mints atoms from `--only`/`--target` values without a bounded/existing-atom check. I did not report this because these values come from the developer's own command line (operator-controlled), not from untrusted network/file input, so the atom-table-exhaustion threat model does not apply through this entry point. Same pattern is used unguarded in `fetcher.ex`, i.e. it is established internal convention, not a divergence introduced here.
- **`--target` is undocumented** in the `@moduledoc` but is accepted by the parser. This is a doc-completeness gap, not a behavioral bug (the switch works), so I did not raise it as a finding.
- I verified the conversion and filtering logic in isolation with `elixir -e`; I did not stand up a full fixture project to run `mix deps.get` end-to-end against a real SCM, as the file's own logic (the only thing in scope) is fully exercised by the isolated checks above.

### Ruled out

- **`check_locked` boolean corrupted by string interpolation** — verified `"#{true}"` → `"true"` → `String.to_unsafe_atom("true")` → `true`, and `Keyword.get` yields boolean `true`; no corruption.
- **Wrong argument order to `Mix.Dep.Fetcher.all/3`** — checked the callee's head `all(old_lock, new_lock, opts)`; the call site matches upstream `deps.get` semantics (empty old_lock, current lock as new_lock).
- **`else :ok` branch / return value** — the arm only affects the (unused) return of `do_run`; the user-visible info message is gated correctly on `apps == []`, confirmed against the test expectation.
- **`with_deps_lock` single-arg dispatch** — confirmed the default-parameter clause binds the closure to `fun`; no arity mismatch.


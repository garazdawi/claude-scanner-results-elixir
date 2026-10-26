# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.start.ex
<!-- sha256: 3adc6c01fe83737119901cbd31ab081ad8a3bd76d925b5ed7cc569b9ae13e491 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:33:24Z_

## Summary

`app.start.ex` is a small, stable Mix task that compiles/configures the project and starts its applications; I found no genuine, reachable bugs in it.

## Bugfixes

No findings

## Risks / caveats

- `start/3` and `type/2` are marked `@doc false` (nominally public but internal); I treated the real entry point as `run/1`. The error-classification in `start/3` (`{:error, {app, reason}} when type == :permanent` vs. the fallback clause) relies on `Application.ensure_all_started/2` returning the `{:error, {App, Reason}}` shape. This shape is stable across the OTP versions this Elixir source targets (including the `mode: :concurrent` path added in OTP 26); I confirmed the code shape but did not exercise a live application-start failure in both `:serial` and `:concurrent` modes to observe the exact error tuple, so that dependency is asserted from OTP's documented contract rather than run.
- The `--no-start` detection uses a raw-args membership test (`"--no-start" in args`) rather than the parsed `opts`. I verified via OptionParser that this is deliberate and robust: `--no-start` is not a declared switch, so it does not surface in `opts`, and the raw-args check is unaffected by OptionParser's negation handling (confirmed `OptionParser.parse(["--permanent","--no-start","--force"], switches: @switches)` yields `[permanent: true, force: true]` — no `start`/`no-start` key). No bug.

### Ruled out

- **`type/2` precedence (`--temporary` + `--permanent` together)** — verified with `OptionParser.parse(["--temporary","--permanent"], ...)` → `[temporary: true, permanent: true]`; the `cond` checks `opts[:temporary]` first, so `:temporary` deterministically wins. This ordering is a defensible, documented-consistent choice, not a defect.
- **Unhandled extra CLI flags (`--force`, `--preload-modules`, `--no-*`)** — these are consumed downstream by `app.config`/`compile`; `run/1` only reads `opts[:temporary]`/`opts[:permanent]` from its own parse, and non-strict `OptionParser` tolerates the rest (verified `--force` parses without error). No crash or misbehavior.
- **`apps/1` returning `[]`** (no `:app`, non-umbrella) — `Application.ensure_all_started([], ...)` returns `{:ok, []}`, matched by the first clause. No `FunctionClauseError`/`CaseClauseError`.
- **`--no-start` skipping start but calling `Mix.Task.reenable("app.start")`** — intentional so a later real `app.start` run is not suppressed by Mix's run-once semantics; matches the documented "does not actually start applications, only compiles and loads code" behavior (compile/load happens via the earlier `app.config` call).
- **`Logger.App.stop()` on a node without Logger** — guarded by `@compile {:no_warn_undefined, Logger.App}` and the moduledoc's note that Mix must not hard-depend on Logger; `Logger.App` is part of the standard distribution loaded before this runs. Not reachable as a crash through the documented API.


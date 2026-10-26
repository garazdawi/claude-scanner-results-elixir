# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell.ex
<!-- sha256: 5c68206c76d8e107746cdc1e8a0f23cb8e81ecf6de126ae702a77343effc895b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:25:18Z_

## Summary

`Mix.Shell` defines the shell behaviour contract and provides two concrete public functions — `printable_app_name/0` (a thin delegate) and `cmd/3` (a streaming command runner) — plus a `Collectable` impl for the `%Mix.Shell{}` struct; the file is small and, after testing every public path, I found no genuine behavioural bugs.

## Bugfixes

No findings

## Risks / caveats

- `printable_app_name/0` merely delegates to `Mix.ProjectStack.printable_app_name/0`, which lives in another file and is out of scope for this review; I did not audit its internal correctness.
- I verified `cmd/3` behaviour by execution (normal callback, `:quiet`, `{executable, args}` tuple form, default `:stderr_to_stdout` redirection, and the `use_stdio: false` path) — all matched the documented contract. The `:quiet` branch installs `fn x -> x end` (identity) rather than a literal no-op; since the streamed return value is discarded by the `Collectable` collector, this is observably a no-op (verified: no output printed under `quiet: true`), so it is not a bug.
- The `Collectable` impl's collector handles `{:cont, data}`, `:done`, and `:halt` (via the `_, _ -> :ok` catch-all) and returns `:ok` as the final accumulator, which `cmd/3` discards via `{_, status}` — consistent with `System.cmd/3`/`System.shell/2` semantics.

### Ruled out

- **`:stderr_to_stdout` default logic (lines 153–159)** — Doc states it defaults to `true` unless `:use_stdio` is `false`. Code sets `use_stdio = Keyword.get(options, :use_stdio, true)` then `Keyword.put_new(:stderr_to_stdout, use_stdio)`, exactly matching the documented default. Verified: `echo err 1>&2` reached the callback via stdout by default (`OUT:err`), and `use_stdio: false` ran without the stderr/stdio conflict `System.cmd` would otherwise raise.
- **Mid-arg default `options \\ []` with the `is_function(callback, 1)` guard (line 145)** — Generates `cmd/2` and `cmd/3`; `cmd/2` correctly fills `options` with `[]`. Verified by calling the 2-arg form successfully.
- **`:quiet` callback override (lines 146–151)** — Replaces callback with identity; return value is ignored so no output is emitted. Verified no output under `quiet: true`.
- **`Keyword.take([:cd, :stderr_to_stdout, :env, :use_stdio])` dropping `:quiet` (line 157)** — Intentional; `:quiet` is consumed earlier by the callback swap, so it must not be forwarded to `System`. Not a defect.


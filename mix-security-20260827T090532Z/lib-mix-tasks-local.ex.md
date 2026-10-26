# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.ex
<!-- sha256: 72755d46ce26df1dc9d27e62cb07962337cd12fd5558edf1ab16f9ee17687922 -->

_Analysed on 2026-08-27T11:12:52Z_

Confirmed. A non-string/non-atom/non-iolist `@shortdoc` term (tuple, plain integer, or list containing a tuple) makes `:io_lib.format` raise `ArgumentError`; atoms (`nil`) and iolists are tolerated. `task_name` returns a binary, so `byte_size(task)` and `~s` are safe. Data arguments are not re-interpreted as directives, and `max` is always an integer, so the format template cannot be injected. Every one of these behaviors sits behind the same trust boundary. I have enough to write the report.

## Summary

`Mix.Tasks.Local.run/1` only formats task-name/`@shortdoc` metadata drawn from already-installed archives into a display line, and every untrusted-data path sits behind the documented full-trust archive-install boundary, so there is no reachable security defect in this file.

## Security

No findings.

The file's single entry point, `run([])` (reachable as `mix local`), pulls its data exclusively from `Mix.Local.archives_tasks()` — modules loaded from *installed* archives — and feeds two values, `task` (task name) and `doc` (`@shortdoc`), into `:io_lib.format/2` at `local.ex:32`. I traced and tested every way that sink could misbehave:

- **Format-string injection — not present.** The template `~c"mix ~-#{max}s # ~ts"` interpolates only `max`, which is computed via `byte_size/1` and is always a non-negative integer (verified: template renders as `~c"mix ~-15s # ~ts"`). `task`/`doc` are passed as *data arguments*, not spliced into the template, so a `~`-laden `@shortdoc` is printed literally (verified: `"~w~n~s hacked"` → emitted verbatim). No second-order directive injection.
- **Sink crash on malformed `@shortdoc` — real but out of scope.** A `@shortdoc` set to a tuple / plain integer / list-containing-a-tuple makes `:io_lib.format` raise `ArgumentError` (verified: `{:evil,1}`, `123`, `[1,{2}]` all crash; `nil`, atoms, and iolists are tolerated). This aborts `mix local`, but the value originates in a module the developer *explicitly installed* via `mix archive.install`, which the documented security model treats as arbitrary-code-execution ("Archives must be installed only from sources you trust"; an installed archive loads as a plugin on every Mix command). A crash is strictly weaker than the RCE such an archive already holds, so it is not a security finding here.

### Ruled out
- **Crash of `mix local` via a non-string `@shortdoc` term (`:io_lib.format` `ArgumentError`)** — reachable only from an installed archive, which is full-trust code execution by design; the crash is self-inflicted and weaker than what the archive can already do. Robustness nit, not a vulnerability.
- **Terminal-escape passthrough** — `doc`/`task` are printed raw via `shell.info`, so ANSI/control bytes in an archive's `@shortdoc` reach the terminal unescaped; same archive-full-trust source (an installed archive can write to the terminal directly during plugin load), so no escalation. (Contrast with unauthenticated-peer escape injection, which this is not.)
- **`max`/module-count resource blowup** — task names derive from module atoms (≤255 chars), and the archive set is bounded by explicit installs; no attacker-controlled amplification.
- **Format-template interpolation of `max`** — `max` is integer-typed via `byte_size/1`; cannot carry directives.
- **No other sink classes present** — this file has no `Code.eval*`/`apply`, no bytes→atom conversion (both helpers operate on already-loaded module atoms), no command execution, no filesystem/path handling, no archive extraction, no deserialization, no network, no crypto, no shared-state writes, no concurrency, and no defensive guards that could be dead.

## Risks / caveats
- The two data-supplying helpers (`Mix.Task.task_name/1`, `Mix.Task.shortdoc/1`) and the loader (`Mix.Local.archives_tasks/0`) live in other files and were only inspected enough to establish that `doc` can be `nil` / an arbitrary `@shortdoc` term and `task` is a binary; their internals were not reviewed (out of scope).
- Reproductions ran against the in-tree build (Elixir 1.21.0-dev / OTP 29). The initial run emitted a latin1 native-encoding warning; I re-ran the crash tests under `ELIXIR_ERL_OPTIONS="+fnu"` (UTF-8) and results were unchanged.
- Conclusions rest on the standing threat-model premise that `mix archive.install` is a full-trust, operator-authorized action. If a deployment somehow populated the archives directory (`MIX_ARCHIVES`) from an untrusted source without an explicit trust decision, the `@shortdoc`-driven crash and terminal-escape passthrough would become live availability/spoofing issues — but that population step, not this file, would be the actual boundary violation.


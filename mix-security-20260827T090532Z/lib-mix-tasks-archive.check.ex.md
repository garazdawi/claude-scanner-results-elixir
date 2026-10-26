# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.check.ex
<!-- sha256: 592235f18f0493e7808761572163d4c6fd6f61ca9d959b698491e30e32aac587 -->

_Analysed on 2026-08-27T10:20:22Z_

All facts confirmed. `Application.load/1` only looks up the `.app` spec (`{:error, {~c"no such file...", ...}}`) and executes no module code; `Version.parse_requirement` rejects junk with `:error`; `Version.Requirement` has a `String.Chars` impl so error interpolation is safe. The task's only input (`:archives`) is read from the root project's `Mix.Project.config()` in every caller — operator-controlled.

## Summary
`Mix.Tasks.Archive.Check.run/1` reads the root project's operator-controlled `:archives` config and checks each named archive's loaded version against a requirement, reaching no dangerous sink — no code execution, atom creation, path handling, extraction, network, or shell — so there are no security findings.

## Security

No findings.

### Ruled out
- **`Application.load(archive)` as a code-execution sink** — `archive` is required to be an atom (`is_atom(archive)` guard) drawn from `Mix.Project.config()[:archives]`, i.e. the root project's own `mix.exs` (operator-trusted; confirmed every caller — `deps.get`, `deps.update`, `deps.compile`, `deps.loadpaths` — invokes it in the root project context). More importantly, `Application.load/1` only reads the `.app` resource spec and does not start the application or run any module code. Verified: `Application.load(:some_nonexistent_app_xyz)` → `{:error, {~c"no such file or directory", ~c"some_nonexistent_app_xyz.app"}}`. No RCE.
- **Atom creation (CWE-770)** — none. The archive name arrives already as an atom (guarded); nothing here calls `String.to_atom`/`Module.concat`/`binary_to_atom` on external bytes.
- **Version-requirement injection / parser crash** — `Version.parse_requirement/1` is a safe parser returning `:error` on malformed input (verified with `"$(rm -rf /)"` → `:error`), which `parse_archive/1` converts to a `Mix.raise`. No shell/regex/eval reachable through it.
- **Error-message interpolation / format string** — `Mix.raise` takes a plain string, not a `~`-style format string, so there is no format-string sink. `#{req}` interpolates a `%Version.Requirement{}` (has a `String.Chars` impl — verified, no crash) and `#{inspect(other)}` uses `inspect`. No injection.
- **Terminal-escape injection via `vsn`** — `vsn` from `Application.spec(archive, :vsn)` (an installed archive's `.app` file) flows into `#{archive}-#{vsn}` in a `Mix.raise` message with control bytes preserved (`List.to_string(~c"1.0.0\e[31mX")` → `"1.0.0\e[31mX"`, verified). But an installed archive already executes arbitrary code on every Mix command by design (threat model: "install only from sources you trust"), so this post-compromise escape into build output is strictly weaker than the RCE the attacker already holds — not a meaningful boundary crossing.
- **DoS / crash on malformed `.app` vsn** — a non-charlist or non-version `vsn` could make `List.to_string/1` or `Version.match?/2` raise, but that value originates from an already-trusted installed archive and yields only a crash; self-inflicted, not attacker-crossing.

## Risks / caveats
- The reproductions were run with the system Elixir at `/opt/elixir` (the repo is not compiled in-tree, so `lib/elixir/ebin` is absent). The stdlib behaviors exercised (`Application.load/1`, `Version.parse_requirement/1`, `String.Chars` for `Version.Requirement`, `List.to_string/1`) are version-stable, so this does not affect the conclusions; I did not execute the task end-to-end against a real installed malicious `.ez` archive.
- Scope was this file only. Whether an installed archive can be planted or its `.app` tampered with, and the safety of `.ez` extraction/traversal, live in `archive.install.ex`/`Mix.Local`/`Mix.Utils` and were not assessed here; this task merely consumes an already-installed archive's version metadata.


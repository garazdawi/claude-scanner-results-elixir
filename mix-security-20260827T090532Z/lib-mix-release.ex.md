# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/release.ex
<!-- sha256: 806609e5f2775c81d539bd8c33e5a0bffc3cc38aeaf100977b6bd5854e9b6c27 -->

_Analysed on 2026-08-27T09:57:26Z_

Reachability confirmed: the calling task (`Mix.Tasks.Release`) invokes `copy_app/2` with no `vsn` sanitization, and no `Version.parse`/validation exists anywhere between the `.app` read (`:file.consult`, line 374) and the path sink (line 836). The same unvalidated `#{name}-#{vsn}` path pattern is even reused at task line 1174 for tar/overlay listing.

I have enough to write the report.

## Summary

`Mix.Release.copy_app/2` interpolates an application's `vsn` — read verbatim from a dependency's `ebin/<app>.app` file — into a filesystem path with no validation, giving a crafted dependency a path-traversal write/delete primitive outside the release tree during `mix release`.

## Security

#### `CSCAN-release-app-vsn-path-traversal`
- **Finding ID** — `CSCAN-release-app-vsn-path-traversal`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/release.ex:836` (`copy_app/2`), value sourced at `:833` from properties loaded by `do_load_app/8` via `:file.consult` at `:374`.
- **Class** — File operations and path handling (path traversal, CWE-22); zip-slip-class via dependency metadata.
- **Impact** — A dependency (or any hand-crafted `ebin/<app>.app`) whose `{vsn, "..."}` contains `/../` sequences steers `copy_app` outside `<release>/lib`. On the first `mix release` the app's `ebin`/`priv`/`include` contents are written to an attacker-named directory anywhere the build user can write (`mkdir_p!` creates the `<app>-<start-of-vsn>` decoy component that enables the traversal). On any subsequent `mix release` run the persisted decoy lets `File.rm_rf!(target_app)` recursively delete an attacker-named directory *outside* the release tree. `vsn` is never validated as a version or path between the `.app` read and the sink. For a pure-Erlang/rebar dependency (or a precompiled `.app` artifact) the malicious `vsn` is purely declarative data — no dependency code has to execute to obtain this file-write/delete primitive on the operator's build machine. The identical `#{name}-#{vsn}` join is also reused in the release task (`tasks/release.ex:1174`) for the tar/overlay file list, so escaped files are also mis-recorded there.
- **Evidence** — `verified`.
  - Traversal reaches real filesystem sinks — write primitive:
    ```
    $ elixir -e 'release = %Mix.Release{path: "/tmp/scan_demo/rel", erts_source: nil, options: [],
        applications: %{evil: [vsn: ~c"0/../../../../PWNED_OUTSIDE", path: "/tmp/scan_demo/src", otp_app?: false]}}
      Mix.Release.copy_app(release, :evil)'
    # find /tmp/PWNED_OUTSIDE -type f  →
    /tmp/PWNED_OUTSIDE/priv/leak.txt
    /tmp/PWNED_OUTSIDE/ebin/mod.beam        # escaped /tmp/scan_demo/rel/lib entirely
    ```
  - Delete primitive on re-run: `important_after_run1: true` then `important_after_run2: false` — `/tmp/PWNED_OUTSIDE/important.txt` recursively removed by `rm_rf!` once the `evil-0` decoy persisted.
  - `Path.join(["/tmp/rel","lib","app-0/../../../../../../tmp/scan_evil"])` → `Path.expand` → `/tmp/scan_evil`, confirming `Path.join` performs no `..` normalization.
- **Suggested fix** — Validate `vsn` (and, defensively, `app`) before building `target_app`: reject any value containing a path separator or `..` (a legitimate OTP `vsn` is `[\w.\-+]+`), or confine the result with `Path.safe_relative/2` against `Path.join(release.path, "lib")` and raise on escape. The same guard should protect the `#{name}-#{vsn}` join in `tasks/release.ex:1175`.

### Ruled out
- **Release-name regex uses `^…$` instead of `\A…\z`** (`from_config!`, line 99): verified `"foo\n" =~ ~r/^[a-z][a-z0-9_]*$/` returns `true`, so a trailing newline bypasses the name check and flows into the release path. Not reported as a finding: the release name is operator-supplied (CLI `mix release NAME`, `:releases`/`:default_release`/`:app` in `mix.exs`) — it never crosses a dependency/network/archive boundary. Still worth tightening to `\A[a-z][a-z0-9_]*\z`.
- **`:file.consult` on dependency `.app` files** (`do_load_app` line 374, `process_app_file` line 904): creates atoms from dependency-controlled bytes (CWE-770) and the hard match `[{:application, ^app, properties}] = terms` crashes on a malformed `.app`. Both are bounded by a dependency that, in the normal Mix flow, already executed code at `deps.compile` time, and the worst outcome is aborting the operator's own build — no privilege crossing beyond the traversal already reported.
- **`strip_beam/2`** (line 938): `:beam_lib.chunks` on a hostile binary returns `{:error, _, _}` rather than crashing, and `:compress` uses `:zlib.gzip` (compression, not decompression — no decompression bomb). The `binary` is caller-forwarded per its `@spec`; no in-file boundary is crossed.
- **`make_cookie` / `random_cookie`** (lines 570, 586): `:crypto.strong_rand_bytes(32)` + `Base.encode32` is a sound CSPRNG cookie; operator-supplied `:cookie` is by design.
- **`erts_data/1` binary branch** (line 280–288): `[_, erts_version] = String.split("-")` MatchErrors on a multi-`-` basename, but `:include_erts` is an operator option, not cross-boundary.

## Risks / caveats
- The `copy_app` traversal's practical severity depends on the dependency type. For an Elixir/Mix dependency, `deps.compile` already runs the dependency's code, so the traversal is not a privilege escalation there — it remains a release-artifact integrity bug. The genuine no-code-execution exploit is a pure-Erlang/rebar dependency or a precompiled `.app` artifact, where `vsn` is declarative data; I verified the mechanism by driving the public `copy_app/2` directly rather than by fetching a live malicious Erlang dep end-to-end through `mix deps.get`.
- I reviewed only this file plus the `copy_app`/`vsn` reachability points in `tasks/release.ex`; I did not audit whether some SCM/compiler upstream of `mix release` independently constrains `.app` `vsn` (I found no such validation in this file or at the task call site). If a stricter `.app` generator is guaranteed for every SCM, real-world reach narrows, but the public `Mix.Release.copy_app/2` sink is unguarded regardless.
- `make_boot_script`/`make_sys_config` drive `:systools`/`:file.consult`/`Config.Provider` over release-internal and operator-config data; I did not find a dependency-controlled value reaching those path/eval sinks, but a deeper trace of `Config.Provider.init`/`Config.Reader.merge` (out of this file) would be needed to fully clear the config-provider path.


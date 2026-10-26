# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix.ex
<!-- sha256: 46953183c70cecd77c32d9ff427de86bdf2f571e031506879eacd2a8c937f1a0 -->

_Analysed on 2026-08-27T09:13:41Z_

I have complete coverage now. Every sink-shaped call in the file has been traced. Let me write the report.

## Summary

`mix.ex` is Mix's top-level facade — global-state accessors, the `Application` callbacks, `Mix.install/2` orchestration, and path helpers — and every security-relevant sink in it draws its input from the developer, the operator's environment, Mix's own cache files, or trusted infrastructure, with all untrusted-data handling (archive extraction, git shell-out, network fetch, lockfile evaluation) delegated to other modules; I found no in-file vulnerability.

## Security

No findings.

### Ruled out

- **`Base.decode64!(data)` on `merge.lock.md5` (line 997) can raise `ArgumentError`** — verified the crash on malformed input, but the file lives inside Mix's own install cache (`install_project_dir`, an MD5-keyed path) and is written by Mix itself with valid base64 (line 1008). It can only hold hostile bytes via a poisoned `MIX_INSTALL_RESTORE_PROJECT_DIR` (operator env var, out of scope) or a prior dependency that already escaped its build sandbox (arbitrary code execution already occurred). Impact is a self-inflicted build crash, not a cross-boundary issue.
- **MD5 as cache key / change-detector (lines 938, 1001)** — MD5 is collision-broken, but here it keys the `Mix.install` cache and detects lockfile changes over developer-controlled inputs (`deps`, `config`, `system_env`, the developer's own lockfile). It is not an integrity control against an attacker, and it is never decoded, so no round-trip bypass exists.
- **Path traversal in `remove_dep`/`remove_leftover_deps` (lines 1098–1110)** — `dep` names come from `File.ls!` of the build/deps dirs; verified that listing entries are always simple basenames (even `weird..name` stays one component with no `/`), so `Path.join` cannot escape the directory. The MD5-derived `cache_id` was also verified to contain no `/`, `.`, or `+` (base64url), so `install_project_dir` cannot traverse.
- **`File.rm_rf!` / `File.cp_r` / `File.cd!` on `install_project_dir` and `restore_dir` (lines 953, 980, 984, 986)** — all operate on Mix's MD5-keyed cache path or the operator-supplied `MIX_INSTALL_RESTORE_PROJECT_DIR`; copying a previous install and then loading its code is the documented, operator-trusted purpose of that variable.
- **Global mutable state setters — `env/1`, `target/1`, `shell/1`, `debug/1`, `Mix.State.put(:installed, …)`, `System.put_env(system_env)`, `Application.put_all_env(config, …)`** — all take developer/operator-supplied values; Mix is single-project-per-VM and the threat model places the operator out of scope.
- **`Code.prepend_path` / `Application.load` in `ensure_application!` (lines 692–693)** — `app` is already an atom (guard, no atom creation) and the prepended `path` comes only from the trusted `Mix.State.builtin_apps()` map; unknown apps raise rather than loading.
- **`stop_apps` leaves `:silence_app_exit` primary logger filter installed if `Application.stop/1` raises (lines 1193–1197, no `try/after`)** — a robustness nit with no security consequence; inputs are trusted app names.
- **`expand_path` on `:config_path` / `:lockfile` atoms (lines 1070–1083)** — resolves only to a `:path` dependency's directory (raises otherwise), which is developer-specified and local; no atom is created (input already an atom).

## Risks / caveats

- The high-value cross-boundary sinks for this folder — archive/zip-slip extraction, `git` argument injection, `--sha512` checksum ordering, TLS-verified fetching, and lockfile evaluation — are **not in this file**. `Mix.install/2` reaches them only indirectly through `Mix.Task.rerun("deps.get" | "deps.loadpaths" | "compile" | "app.config" | "loadconfig")` and `Mix.Dep.Lock.read/1`, which live in `Mix.Tasks.Deps.*`, `Mix.SCM.*`, `Mix.Utils`, and `Mix.Dep.Lock`. Those require their own review; nothing here weakens or bypasses them.
- `Mix.install/2`'s `deps`/`opts` are, by design, developer-authored code paths (scripts, Livebook notebooks). Running such a script is already arbitrary code execution, so I treated those arguments as trusted; there is no documented Mix entry point in this file that feeds *non-developer* data into `deps` or the path-bearing options.
- I verified behavior against Elixir 1.21.0-dev / OTP 29 built in this tree (`base64url` charset, `decode64!` crash, `File.ls!` basename invariant). I did not exercise the full `Mix.install/2` end-to-end (network/deps resolution) since its sinks are delegated out of this file.


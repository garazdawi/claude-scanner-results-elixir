# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/loader.ex
<!-- sha256: 3718bd22d7fa49705acd84e2c452e45e19da7203366c1a6e577b1b1ecb0ef848 -->

_Analysed on 2026-08-27T09:41:35Z_

I have a definitive end-to-end reproduction. Let me confirm my understanding of the `with_scm_and_app` sink lines and finalize.

The chain is fully verified: unsanitized dependency app name → `Path.join(deps_path, bin_app)` in `with_scm_and_app` (loader.ex:182) → `opts[:dest]`/`:checkout` → `File.rm_rf!` + git write in the SCM, escaping the project. Here is the report.

## Summary

`Mix.Dep.Loader` builds a dependency's on-disk `dest`/`build` paths by joining the deps directory with the raw dependency name (`Path.join(deps_path, Atom.to_string(app))`) with no confinement, so a `..`-laden name coming from an untrusted transitive `rebar.config` lets `mix deps.get` delete and write files outside the project tree.

## Security

#### `CSCAN-loader-dep-app-name-traversal`
- **Finding ID** — `CSCAN-loader-dep-app-name-traversal`
- **Severity** — High
- **Confidence** — high
- **Location** — `lib/mix/lib/mix/dep/loader.ex:182-183` (`with_scm_and_app/5`; same root cause at `validate_app/1` `Path.expand("ebin/#{app}.app", ...)`, line 407)
- **Class** — File operations and path handling (CWE-22 path traversal)
- **Impact** — A project that (transitively) pulls a malicious rebar-managed dependency is compromised at **`mix deps.get`** time — before any compilation. The dependency's `rebar.config` is parsed as *data*, but its declared sub-dependency names become filesystem paths here with no `..`/segment validation. A name such as `'../../PWNED_SENTINEL'` yields `dest = deps/../../PWNED_SENTINEL`, which the Git SCM then feeds to `File.rm_rf!(opts[:checkout])` (git.ex:108) followed by `File.mkdir_p!` + `git init`. Result: **arbitrary directory/file deletion outside the project** (e.g. a sibling checkout, `~/.ssh`, build caches) plus planting attacker-controlled content there. This runs during the fetch phase, which is invoked far more freely than compilation (CI checkout, inspecting deps), and the deletion needs only that a `deps/` directory already exist — the universal case for any project with other deps or a prior `deps.get`.
- **Evidence** — `verified`. Malicious `rebar.config`: `{deps, [{'../../PWNED_SENTINEL', {git, "https://invalid.invalid/never-reached.git", {branch,"main"}}}]}.` in a path-dep rebar package; victim `deps/` pre-existing. Running `mix deps.get`:
  ```
  * Getting ../../PWNED_SENTINEL (https://invalid.invalid/never-reached.git - main)
  ...
  === AFTER ===
  >>> important.txt DELETED
  >>> keys.dat DELETED           # both files in /tmp/PWNED_SENTINEL (outside the project) gone
  ```
  Component check: `:file.consult` on the rebar.config yields the atom `:"../../../../tmp/pwned_dep"`; `Path.join("/home/victim/project/deps", "../../../../tmp/pwned_dep") |> Path.expand` → `"/tmp/pwned_dep"` (escapes the deps root). No app-name validation exists in the loader flow (`grep` for `safe_relative`/name checks → none).
- **Suggested fix** — Before building `dest`/`build`, validate that `bin_app` is a single safe path segment, or confine it: reject names where `Path.safe_relative(bin_app)` returns `:error` (or where `bin_app != Path.basename(bin_app)`), raising `invalid_dep_format`/a Mix error. Equivalently, enforce that dependency app atoms match the conventional `~r/\A[a-z]\w*\z/` before they are ever used as a path component.

#### `CSCAN-loader-app-vsn-iodata-crash`
- **Finding ID** — `CSCAN-loader-app-vsn-iodata-crash`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/mix/lib/mix/dep/loader.ex:465` (`app_status/3`, the `{:vsn, actual} when is_list(actual)` clause)
- **Class** — Resource consumption / robustness (uncaught crash on untrusted input; defeats intended graceful handling)
- **Impact** — The `:vsn` field is read straight from a dependency's shipped `.app` file. The `is_list(actual)` branch calls `IO.iodata_to_binary(actual)` with no guard; a dependency shipping a list that is not valid iodata (e.g. `{vsn, [1114112]}`) raises `ArgumentError` uncaught, aborting `mix deps.get`/`compile` with a raw stacktrace instead of the clearly-intended graceful `{:invalidvsn, actual}` status that the sibling clause produces. A malicious/broken dependency can thus hard-crash the build tool. Low impact (denial of a clean error, not compromise), but it bypasses the explicit invalid-version handling right next to it.
- **Evidence** — `verified`. `IO.iodata_to_binary([1_114_112])` → `ArgumentError - ... 1st argument: not an iodata term`.
- **Suggested fix** — Wrap the conversion (`try/rescue ArgumentError -> {:invalidvsn, actual}`) or validate with a total function, so a malformed charlist `vsn` yields `{:invalidvsn, actual}` like every other bad-version case rather than crashing.

### Ruled out
- **`vsn_match/3` regex `actual =~ req` / `Mix.raise("Invalid requirement #{req}...")`** — the `Regex`/requirement `req` originates from the *operator's own* `mix.exs` (or a parent dep the operator chose); `actual` is a plain version string fed to `Version.parse`. No attacker-controlled pattern reaches `Regex.compile`, so no ReDoS/injection across a trust boundary.
- **`with_system_env/2` → `System.put_env(new_env)`** — `system_env` is only populated from the `:system_env` dependency option in `with_scm_and_app`; the rebar-config parser (`Mix.Rebar.parse_dep`) never emits `:system_env`, and `attach_only_and_targets` only propagates `:only`/`:targets`. It is therefore reachable only from operator-authored `mix.exs`, not from an untrusted dependency.
- **`app_status/3` `Path.wildcard(Path.join(dir, "*.app"))`** — the glob pattern is the literal `"*.app"`; only the directory is variable, and that variability is the same app-name issue already covered by the traversal finding, not an independent wildcard-injection cost sink.
- **`skip?/2`** — iterates `List.wrap` and raises on non-atom entries; no sink, only a `Mix.raise` on malformed operator config.

## Risks / caveats
- The traversal detonates in `Mix.SCM.Git.checkout/1` (`File.rm_rf!` at git.ex:108) and `File.mkdir_p!`, which are outside the file under review; I attribute the finding to `with_scm_and_app` because that is where the unconfined `Path.join` builds the path and where the missing `Path.safe_relative`/segment check belongs (git.ex reasonably trusts `opts[:dest]` as Mix-internal). The same traversing `dest` also feeds the Path/Hex SCMs; I only exercised the Git SCM.
- I reproduced the destructive delete with the sub-dep's Git URL deliberately unresolvable, because `File.rm_rf!`/`mkdir_p!` run *before* the network fetch; with a reachable attacker repo the same run would also plant an arbitrary checkout at the traversed location. I did not exercise a live remote fetch.
- App names arriving via the Hex remote converger (`mix_dep/3` second clause → `to_dep`) flow through the identical sink; Hex validates package names upstream, so I anchored the reproduction on the rebar path where the name is unvalidated data. I did not audit every SCM's `accepts_options` for additional normalization that might blunt the traversal for non-Git sources.


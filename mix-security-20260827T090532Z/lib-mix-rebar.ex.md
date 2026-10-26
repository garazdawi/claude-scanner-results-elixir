# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/rebar.ex
<!-- sha256: a71a025d83a410fced1f4679eb36a802fde701effe4ab0d3d41b7fcfce7dc766 -->

_Analysed on 2026-08-27T09:56:47Z_

I have enough to write the report. Summary of what I verified:

- **Trust boundary confirmed:** `Mix.Rebar.load_config/1` and `deps/1` run on a *fetched dependency's* `rebar.config` / `rebar.config.script` during dep-tree loading (`Mix.Dep.Loader.rebar_dep` → `mix deps.get` / `deps.compile` / `compile`).
- **`load_config` evaluates `rebar.config.script`** as arbitrary Erlang (verified: marker file + env var written) — this is the documented, by-design rebar-compat behavior.
- **`compile_req/1` compiles a dependency-controlled requirement string into a `Regex`**, later matched with `=~` in `vsn_match/3` (verified end-to-end), but OTP's `:re` `match_limit` caps each match at ~76 ms regardless of input size.
- Malformed config tuples crash the load (`MatchError`/`FunctionClauseError`) — self-inflicted build DoS.

## Summary

`Mix.Rebar` reads and evaluates a fetched dependency's `rebar.config`/`rebar.config.script`; the only unintended (non-by-design) sink is a dependency-controlled version string compiled into a regex, whose blowup OTP's `:re` engine already bounds, leaving no high-impact vulnerability in this file.

## Security

#### `CSCAN-rebar-requirement-regex-redos`
- **Finding ID** — `CSCAN-rebar-requirement-regex-redos`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/mix/rebar.ex:198`, `compile_req/1` (reached from the public `deps/1`, line 124); the compiled pattern is executed downstream at `Mix.Dep.Loader.vsn_match/3` (`lib/mix/dep/loader.ex:125`, `actual =~ req`).
- **Class** — Resource consumption (regex pattern injection / ReDoS, CWE-1333).
- **Impact** — A dependency's `rebar.config` can declare a dep whose "requirement" is any string. When it does not parse as a `Version` requirement, `compile_req/1` compiles it verbatim as a `Regex` (no ReDoS guard), and that regex becomes the Mix dependency's `requirement`. During convergence, `vsn_match/3` runs it as `actual =~ req` against a version string the same attacker controls (the dependency's own `.app` version). A pathological pattern (`^(a+)+$`) forces catastrophic backtracking. Impact is bounded: OTP's `:re` `match_limit` caps each match at ~tens of milliseconds even for 100 KB inputs, so this is a modest per-match slowdown during `mix deps.get`/`compile`, not an unbounded hang. It is strictly weaker than the code execution the same config already grants via `rebar.config.script` (see caveats), so it only matters for a dependency that ships a hostile `rebar.config` but no `.script`.
- **Evidence** — `verified`. `Mix.Rebar.deps([deps: [{:foo, ~c"^(a+)+$", {:git, ~c"https://example.com/foo.git", {:branch, ~c"main"}}}]])` returns `{:foo, ~r/^(a+)+$/, [override: true, git: "...", branch: "main"]}`; `Mix.Dep.Loader.vsn_match(re, actual, :foo)` runs the pattern. Timing an isolated match: `n=20 → 20.6 ms`, `n=25 → 77 ms`, and it plateaus (`n=100000 → 76.6 ms`), showing `:re`'s `match_limit` caps the backtracking rather than allowing an exponential hang.
- **Suggested fix** — Do not compile an arbitrary dependency-supplied string into a regex. For rebar3-sourced deps a non-parseable version string is not a legitimate requirement; reject it (or drop the requirement) instead of falling back to `Regex.compile/2`. If the regex-requirement translation must be kept, restrict it to a safe subset or run the eventual `=~` under an explicit `:re` timeout/lower `match_limit`.

### Ruled out
- **`serialize_config/1` (`:io_lib.print`, line 102)** — output-only; faithfully re-serializes terms that already came from the dependency, adding no capability the dependency did not already have.
- **`rebar_args/2`, `local_rebar_path/1`, `env_rebar_path/1`** — the rebar executable path derives from `MIX_REBAR3` and `System.version()`/`otp_release()` (operator/VM state), not from attacker-controlled data.
- **`dependency_config/1`** — strips `project_plugins` and `warnings_as_errors`; it is a hardening/normalization step with no sink.
- **`apply_overrides/3`** — pure `Keyword` manipulation of the config's own `overrides`; malformed entries only crash (folded into the malformed-config caveat), no injection into a downstream interpreter.

## Risks / caveats

- **`rebar.config.script` evaluation is arbitrary code execution on the build machine — by design, so not reported as a finding.** `load_config/1` → `eval_script/2` calls `:file.script/2` (line 213) inside `File.cd!(dir, …)`, evaluating the dependency's script with no checksum/signature gate. Verified: a planted `rebar.config.script` wrote `/tmp/rebar_demo/PWNED` and set an env var merely from `Mix.Rebar.load_config/1`. This fires during dependency *loading/convergence* (any command that loads the dep tree, including `mix deps.get`), for transitive rebar deps too — a supply-chain surface a developer may underestimate. It is nonetheless the documented purpose of the function and matches rebar3's own behavior; the folder threat model explicitly places "dependency build scripts run arbitrary code" out of scope, so it is a caveat, not a vulnerability. There is no in-file mitigation to suggest short of not supporting rebar dependencies.
- **Atom-table growth via `:file.consult/1` (line 79, CWE-770).** Reading a dependency-controlled `rebar.config` parses Erlang terms, interning every atom literal; a config with ~1M distinct atoms could exhaust the (non-GC'd) atom table and abort the build. This is a data-only path (no `.script` needed) but inherent to reading Erlang term files the way rebar does — there is no safe drop-in replacement that preserves functionality, and it is overshadowed by the intended script-eval RCE. Not reported as a separate finding for those reasons.
- **Malformed-config crashes = self-inflicted build DoS.** `Mix.Rebar.deps/1` raises `MatchError` on a 1-element source tuple (`{:foo, {:git}}` → `[scm, url | _]` fails, line 169) and `FunctionClauseError` when a version requirement is a binary rather than a charlist (`List.to_string/1`, line 191) — both verified. A hostile dependency can thus abort the consumer's `mix deps.get`, but this is immediately visible and strictly weaker than the code-exec the same config already permits.
- **Untrusted git URL/ref forwarded downstream (sink out of this file).** `parse_source/1` (lines 168–188) passes the dependency-controlled git `url`, `ref`, `tag`, `branch`, and `sparse` values into Mix's `[git: …, ref: …]` options. Whether any of these can inject arguments into the underlying `git` invocation is a property of `Mix.SCM.Git` (`System.cmd("git", …)`), which is outside the file under review; I did not assess it here.
- **`File.cd!/2` mutates the VM-global working directory** for the duration of the script eval (line 212). Any concurrent BEAM process observes the changed cwd; however, the code inside is already fully attacker-controlled by design, so this adds no capability beyond the script eval itself. Noted for completeness.


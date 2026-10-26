# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/cover_record.exs
<!-- sha256: a9d32196f9f64b19aed90c3be792d3ea65abf11251c9fe9f15f4da96d843db62 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:55:39Z_

Confirmed: the sole real caller passes the compile-time literal `"elixir"` (sibling apps pass their own literals like `"eex"`, `"mix"`); there is no dynamic derivation from argv, env, network, or a dependency. I have enough to write the report.

---

# Security review — `cover_record.exs`

## 1. Summary

`CoverageRecorder` is a `COVER=1`-gated, CI-only helper that cover-compiles Elixir's *own* build artifacts and exports coverage data; every sink I traced is fed by compile-time literals or Elixir's own build tree, so nothing crosses an untrusted trust boundary and there are **no actionable findings**.

## 2. Security

`No findings.`

All security-relevant sinks in the file were traced to trusted inputs (hardcoded suite-name literals; `.beam` files produced by Elixir's own compilation under `__DIR__/../../..`). The module is required only by Elixir's internal test harness and `cover.exs`, runs only when the operator sets `COVER∈{1,true,yes,y}`, and never executes during `mix deps.get` / `mix compile` / `mix archive.install` of a downstream project. No data from the network, disk-as-attacker-surface, a dependency, or an untrusted build reaches any sink.

### Ruled out

- **Path traversal via `suite_name` interpolation (line 40, `record/1`).** `file = Path.join(cover_dir(), "ex_unit_#{suite_name}.coverdata")` is unsanitized and reaches `:cover.export` (file write) and `File.mkdir_p!`. I *verified* the traversal mechanism works — `Path.join("/build/cover", "ex_unit_../../../../tmp/pwned.coverdata")` expands to `/tmp/pwned.coverdata` — so this would be an arbitrary-file-write primitive **if** `suite_name` were attacker-controlled. It is not: the documented public entry point `maybe_record/1` is called only with compile-time string literals (`maybe_record("elixir")` in `test/elixir/test_helper.exs:129`; sibling apps pass their own literals `"eex"`/`"ex_unit"`/`"iex"`/`"logger"`/`"mix"`). There is no data flow from argv/env/network/dependency into `suite_name`. Reaching the sink therefore requires a caller to deliberately pass a hostile literal, i.e. misuse of the interface with a value that violates the parameter's implied contract — excluded by scope. Basis: **verified** (traversal reproduced; sole in-tree caller read).
- **Unbounded atom creation via `String.to_unsafe_atom/1` on beam basenames (line 88, `skip_from_coverage?/1`).** `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(string, :utf8)` (verified in `lib/string.ex:3040-3042`; its own docs warn the input "should not be an untrusted value") — an unbounded, non-GC'd atom sink (CWE-770). The input is `Path.basename(file, ".beam")` where `file` comes from `Path.wildcard("lib/*/ebin/*.beam")` under `root_dir()` — i.e. the module names Elixir itself just compiled. Those atoms already exist as loaded/loadable Elixir modules, so `binary_to_atom` returns existing atoms and does not grow the table in practice; producing novel names requires writing `.beam` files into Elixir's own `ebin/`, which already grants code execution (those same beams are then `:cover.compile_beam`'d and loaded). The atom sink is strictly weaker than the write access it presupposes, and the count is bounded by the module set. Basis: **verified** mechanism; boundary by reasoning.
- **Beam loading/instrumentation via `:cover.compile_beam/1` (line 54).** Loads and re-instruments the same `lib/*/ebin/*.beam` files. Controlling them = write access to Elixir's build tree = pre-existing code execution; not a boundary this script defends. Basis: reasoned.
- **`Code.fetch_docs(mod)` (line 89).** Reads the `Docs` chunk from the beam/loaded module for a `deprecated`-metadata match; it does not evaluate or apply module code, so it is not a code-execution sink. `mod` is the bounded, trusted module atom from the previous line. Basis: reasoned from API semantics.
- **File write in `System.at_exit` callback — `File.mkdir_p!(cover_dir())` + `:cover.export` (lines 44-48).** Target directory `cover_dir()` = `root_dir()/cover` is fixed (no external component); the only variable component is `suite_name` (ruled out above). No exploitable TOCTOU: both the mkdir and export target Elixir's own tree, writable only by whoever already runs the suite. Basis: reasoned.
- **`Path.wildcard/1` (lines 37, 67).** Patterns (`lib/*/ebin`, `*.beam`) are constant and the base is `__DIR__`; neither pattern nor cost is externally influenced. Basis: reasoned.
- **`enabled?/0` env read (lines 26-34).** `System.fetch_env("COVER")` is compared against a fixed allow-list (`~w[1 true yes y]`); it gates behavior on a boolean only and cannot inject a value into any sink. Basis: reasoned.
- **Dead defensive code.** None. The `enabled?` gate, the `{:error, reason} -> raise` arm of `cover_compile_ebins/0`, and the `docs_v1`/`deprecated` `match?` filter are all reachable on normal inputs.

## 3. Risks / caveats

- **Sibling call sites not in this extraction.** Only the `elixir` app's tree was available; the `eex`/`ex_unit`/`iex`/`logger`/`mix` `test_helper.exs` callers of `maybe_record/1` were not read directly. The overwhelming convention (and the one caller I did read) is a bare string literal; a hypothetical caller deriving `suite_name` from `System.argv`/env would turn the ruled-out line-40 traversal into a live arbitrary-file-write. This is the one residual worth a defense-in-depth note.
- **Defense-in-depth suggestion (not a finding).** Since `suite_name` is only ever a short identifier, constraining it before interpolation — e.g. `Path.basename("ex_unit_#{suite_name}.coverdata")` or a `String.replace(suite_name, ~r/[^a-z_]/, "")` allow-list — would make the traversal unreachable regardless of future callers, at zero cost to current behavior.
- **Trust model.** The entire assessment rests on this script being Elixir-repo-internal CI tooling operating over Elixir's own compiled artifacts, gated behind an operator-set `COVER` variable. It is not part of the shipped standard library and does not run in a downstream project's build/run path. If that assumption were violated (e.g. the module were repackaged and driven by untrusted suite names / an untrusted `ebin/`), the ruled-out atom-creation and path-traversal items would need re-rating.


# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/format.ex
<!-- sha256: a47b2164c377912edb0efe89b943f79e72b503d3f598a14c04241c9d3231376b -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/format.ex
<!-- sha256: a47b2164c377912edb0efe89b943f79e72b503d3f598a14c04241c9d3231376b -->

_Re-analysed on 2026-08-27_

Both findings re-reproduced end-to-end this session. The three first-pass hedges have been resolved by trying the axes they named: (1) the grapheme-level `String.myers_difference` path was pushed to worst case in isolation and its real-formatter reachability was probed; (2) the manifest `binary_to_term` atom-minting axis and its `[:safe]`-rejection counterfactual were both run in a fresh VM; (3) the non-list-`sources` crash was executed. No callgraph-neighbour read was substituted for an executed axis.

## Summary

`mix format` is largely governed by the standard Mix trust model (operator-controlled `.formatter.exs` is executed as code by design), but its `--check-formatted` diff renderer contains an uncapped quadratic Myers-diff over attacker-influenceable file content, and its config manifest is deserialized with an unsafe `binary_to_term`.

## Security

#### `CSCAN-format-check-formatted-myers-quadratic`
- **Finding ID** — `CSCAN-format-check-formatted-myers-quadratic`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/mix/tasks/format.ex:904` (`text_diff_format/3` → `List.myers_difference/2`) and `:1049` (`line_diff/3` → `String.myers_difference/2`), reached via `run/1` → `check!/2` (`:848`) → `to_diffs/1` (`:867`).
- **Class** — Resource consumption (quadratic time + memory DoS).
- **Impact** — `mix format --check-formatted` is documented (`:68`–`:73`) as the way to "reject contributions with unformatted code" in "pre-commit hooks and CI scripts" — i.e. it is *intended* to run over untrusted contributions. When any input file is "not formatted", `to_diffs/1` unconditionally builds a line diff with `List.myers_difference/2` (`:904`) and, for a single changed line, a grapheme diff with `String.myers_difference/2` (`:1049`). Myers is O(N·D) in both time and space; a file whose formatted output shares no lines with the input drives D≈2N, i.e. O(N²). An attacker who can add a plausible-looking source file under an already-matched input pattern (e.g. a fork PR in CI) can make a sub-1 MB file consume tens of seconds of CPU and multiple GB of RAM, hanging or OOM-killing the job instead of being cleanly rejected. File content is only *parsed*, never executed, so this is a content-only vector that applies even when `mix.exs`/`.formatter.exs` are trusted, and there is no size cap anywhere on the path (both the raising `Mix.raise` and the `--no-exit` branch of `check!/2` compute the diff before printing).
- **Evidence** — `verified`, re-reproduced this session on the checkout (OTP 29 / Elixir 1.21.0-dev).
  - **Line-level path, real `mix format --check-formatted`** end-to-end in a scratch Mix project (`inputs: ["*.ex"]`), input = N lines each with trailing whitespace the formatter strips (every line changes → line-disjoint del/ins): `N=2000 → 1013 ms`, `N=4000 → 4110 ms`, `N=8000 → 23597 ms` — ≈4–5.7× per doubling, super-linear. All inputs well under 1 MB.
  - **Grapheme-level path, tried to worst case in isolation** (resolving the first-pass hedge): calling `Mix.Tasks.Format.text_diff_format/3` directly with a single N-char line whose del/ins share no common subsequence (`String.duplicate("a",N)` vs `String.duplicate("b",N)`, forcing D=2N): `N=2000 → 1.46s`, `N=4000 → 7.06s`, `N=8000 → 25.5s` — confirmed independently quadratic. A near-identical single long line (`"abab…"` vs `"baba…"`, small D) returned in ~0.0s, confirming the cost is driven by edit distance, not length alone.
- **Reachability note (grapheme path, hedge resolved by trying the axis)** — I probed whether the real formatter can emit a multi-thousand-char *changed* line that also has large myers distance (the input needed to make the grapheme path dominate). It cannot in the worst case: (a) a long *wrappable* expression is reflowed by `Code.format_string!` into many short lines (measured: a 7896-byte single line → 118 output lines), which feeds the **line-level** path, not a single long grapheme diff; (b) a *monolithic* long string literal has its 4000-char body preserved, so del/ins share that entire substring and D stays small (fast). The grapheme worst-case is therefore effectively unreachable through `mix format`; the fully-reachable and sufficient amplifier is the line-level `List.myers_difference/2`, which the end-to-end numbers above exercise. Both paths share the same missing-cap root cause and fix, so the finding stands on the line-level path regardless of grapheme reachability.
- **Suggested fix** — Cap the diff work before calling Myers: skip (or truncate to a bounded window) `text_diff_format` when the combined line count or byte size exceeds a threshold, and fall back to printing just the file name / a "file differs, too large to diff" notice. Bounding `max(length(old), length(new))` (and the per-line length fed to `line_diff`) removes the quadratic amplification while preserving the human-readable diff for normal cases.

#### `CSCAN-format-manifest-binary-to-term-unsafe`
- **Finding ID** — `CSCAN-format-manifest-binary-to-term-unsafe`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/mix/tasks/format.ex:538`, `non_raising_binary_to_term/1`; reached from `read_manifest/1` (`:526`–`:535`) via `eval_deps_and_subdirectories/5` → `maybe_cache_in_manifest/3` (`:521`).
- **Class** — Deserialisation (CWE-502 / CWE-770 atom-table exhaustion).
- **Impact** — The cached formatter manifest (`Mix.Project.manifest_path()/cached_dot_formatter`) is decoded with `:erlang.binary_to_term(binary)` and **no `[:safe]` flag** (`:538`). A crafted manifest mints arbitrary new atoms (never garbage-collected) and instantiates arbitrary term structures. Additionally the decoded `sources` flows straight into `Enum.flat_map(sources, &Path.wildcard/2)` in `read_manifest/1` (`:529`), which is *outside* the `rescue` in `non_raising_binary_to_term/1` (that `rescue` guards only the decode call itself), so a non-list `sources` crashes `mix format` unrescued. The trust boundary is weak — the manifest lives in the local `_build` tree — so exploitation requires an attacker who can write `_build` (a shared/restored CI `_build` cache, a committed `_build`, or a co-tenant). The decoded value is treated as data (a `locals_without_parens` list, not executed), so impact is limited to atom-table pollution and resource/DoS rather than code execution — hence Low.
- **Evidence** — `verified`, all three axes run this session in fresh VMs (OTP 29):
  - **Unsafe minting:** a manifest-shaped term `{2, entry, sources}` carrying 20000 runtime-generated atoms, written with `term_to_binary` and decoded in a *fresh* VM exactly as `non_raising_binary_to_term/1` does → `minted 20000 new atoms`.
  - **`[:safe]` counterfactual:** the identical bytes decoded with `:erlang.binary_to_term(bin, [:safe])` as the *only* decode in a fresh VM (no prior interning) → `{:rejected, ArgumentError}`. (An earlier run that decoded unsafe-then-safe in one process showed `[:safe]` succeeding — because the unsafe pass had already interned the atoms; isolating `[:safe]` in its own VM confirms it genuinely rejects.)
  - **Non-list `sources` crash:** decoding `{2, entry, :not_a_list}` (passes the `{@manifest_vsn, entry, sources}` match) then running the `Enum.flat_map(sources, &Path.wildcard/2)` step → `{:UNRESCUED_CRASH, Protocol.UndefinedError}`, confirming the flat_map is not covered by the decode `rescue`.
- **Suggested fix** — Decode with `:erlang.binary_to_term(binary, [:safe])` (already inside the `rescue`, so a rejected payload cleanly degrades to a cache miss), and treat a non-list `sources`/`entry` as `:error` rather than letting `Enum.flat_map/2` raise.

### Ruled out
- **`Code.eval_file/2` on `.formatter.exs` (`:614`, `eval_file_with_keyword_list/1`) for cwd / `--dot-formatter` / `:subdirectories` / `:import_deps` deps** — executes arbitrary code, but every path is gated behind operator-controlled project config or a dependency the operator explicitly listed in `:import_deps`. Per the folder threat model this is by-design (config and dependency scripts run as code; loading a Mix project already evaluates `mix.exs`). Worth noting as a footgun only: `import_deps` reads a dependency's *config* by `Code.eval_file`, so it is code execution, not data import — but it crosses no boundary the existing dep-compile trust doesn't already cross. Confidence: high.
- **`File.write!(file, output)` (`:814`)** — writes only formatter output back to files matched by operator-supplied inputs; non-Elixir files with no plugin format to identity (`input == output` → no write at `:810`), so it is not a general arbitrary-write primitive. Confidence: high.
- **Plugin module dispatch `plugin.format/2` / `plugin.features/1` (`:710`, `:728`, `:775`)** — modules come from operator `:plugins` config; `mix compile` of the project already runs their code. Confidence: high.
- **`colorize/4` regex `~r/[\t\s]+/` (`:1064`)** — a fixed, non-interpolated, linear character-class pattern; not ReDoS. Confidence: high.
- **`Task.async_stream(..., timeout: :infinity)` (`:286`)** — a hanging format blocks indefinitely, but only via an operator-trusted plugin or the (separately-covered) formatter/parser; not a boundary crossing in this file. Confidence: high.
- **Terminal-escape sequences in filenames/content echoed by `Mix.shell()`/diff output (`:817`, `:839`, `:844`, `to_diffs/1`)** — universal to diff/reporting tools and limited to the operator's own working tree; marginal. Confidence: medium.

## Risks / caveats
- The `--check-formatted` DoS severity depends on deployment. In a naive CI that checks out an entire fork PR and runs any `mix` task, `mix.exs` evaluation is a strictly larger problem (RCE) that dominates. The Medium rating reflects the setups the flag is explicitly built for — trusted config with untrusted *content* (pre-commit hooks, config-from-base / source-from-PR splits, snippet-formatting services) — where the quadratic diff is the operative risk. The mechanism and its cost are verified end-to-end this session (real `mix format --check-formatted`, 23.6s at N=8000 on a sub-1MB file); the Medium value is a deployment-context judgment, not an unverified mechanism claim.
- The manifest deserialization is only as reachable as `_build` is writable/shareable in a given environment. The atom-minting and `[:safe]`-rejection axes are verified; the enumeration of concrete `_build`-cache-poisoning chains is a deployment judgment left open. It is consistent with other Mix manifests (which also use plain `binary_to_term`), so it may be considered acceptable project-wide convention rather than a per-file defect — which is why it is rated Low.

## Did I verify this?
- **Line-level quadratic (primary):** Yes — executed real `mix format --check-formatted` end-to-end on a scratch Mix project, 1.0s→4.1s→23.6s across N=2000/4000/8000.
- **Grapheme-level path (first-pass hedge "did not exhaustively search…"):** Axis tried, not asserted. Pushed `text_diff_format/3` to worst case in isolation (25.5s at N=8000, confirmed edit-distance-driven), and probed real-formatter reachability (long lines wrap to multi-line → line path; monolithic literals preserve their body → small D). Conclusion: grapheme worst-case not reachable through `mix format`; line path dominates and is fully reachable. Hedge removed.
- **Manifest unsafe `binary_to_term` (both axes):** Yes — fresh-VM decode minted 20000 atoms; identical bytes under `[:safe]`, isolated in its own fresh VM, raised `ArgumentError`.
- **Non-list `sources` crash:** Yes — executed the decode-match-then-flat_map flow; `Enum.flat_map` raised `Protocol.UndefinedError` outside the decode `rescue`.

### Dropped in re-review
None. Both first-pass findings are preserved and re-verified; no finding was downgraded or removed.

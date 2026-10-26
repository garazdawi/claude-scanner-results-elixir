# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/filters.ex
<!-- sha256: d4582490adf5fbfc759cd0ee15d6ffa04c99ef7be16f95c7875ec167a5405213 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/filters.ex
<!-- sha256: d4582490adf5fbfc759cd0ee15d6ffa04c99ef7be16f95c7875ec167a5405213 -->

_Re-analysed on 2026-08-27 (hedge-resolution pass). Installed `/opt/elixir/.../filters.ex` confirmed byte-identical to the reviewed file (`diff -q` → IDENTICAL), so all runs below are faithful to the reviewed source._

## Summary

`ExUnit.Filters` parses `mix test` command-line filter/path arguments (`--include`/`--exclude`/`--only`, `path:line`, `location:`) and evaluates include/exclude/skip filters against per-test tags to decide which tests run. Every input reaching this file originates from the developer's own `mix test` invocation, test source, or config. The re-review resolved all first-pass hedges by (a) reading the sole in-tree caller (`runner.ex`) and the cross-file deserialization sink (`failures_manifest.ex`), and (b) executing every axis on the ruled-out crash/mis-select entries. One Low finding stands (checklist-mandated atom creation); nothing upgraded to a new finding.

One trust-boundary hedge could **not** be fully cleared: the callers of `parse/1`, `parse_paths/1`, `failure_info/1`, and `fail_all!/1` live in `mix/tasks/test.ex`, which is **not present in this tree** (only `lib/ex_unit/` is checked out). That residual is already captured by the standing Low finding and is restated honestly as a caveat rather than asserted as verified.

## Security

#### `CSCAN-filters-cli-key-atom-creation`
- **Finding ID** — `CSCAN-filters-cli-key-atom-creation`
- **Severity** — Low
- **Confidence** — high
- **Location** — `filters.ex:137-138`, `ExUnit.Filters.parse/1` (both `parse_kv(String.to_unsafe_atom(key), value)` and the bare `String.to_unsafe_atom(key)` branch)
- **Issue** — `parse/1` converts every filter *key* from a raw string to an atom via `String.to_unsafe_atom/1` (= `:erlang.binary_to_atom(string, :utf8)`), an unchecked non-existing-atom creation. The atom table is never GC'd and is capped (default `1_048_576`); each distinct key mints a permanent atom.
- **Class** — Atom creation (CWE-770).
- **Impact** — Unbounded atom-table growth. The `parse/1`/`parse_paths/1` call path is **not present in this tree** — within `lib/ex_unit/` the only callers are `runner.ex` (`normalize/2` @97, `eval/4` @298), and `parse`/`parse_paths`/`failure_info`/`fail_all!` have **no in-tree caller at all**. Per the standing threat model their caller is `mix/tasks/test.ex` (developer CLI), which I could not read here to confirm. If that assumption holds the exhaustion is self-inflicted (Low); the residual is that `parse/1` is documented-public, so any wrapper feeding it externally-sourced strings inherits the sink.
- **Evidence** — `verified` (executed). `elixir` run: `ExUnit.Filters.parse(["a:b:c"])` → `[a: "b:c"]`, and `parse(["foo"])` → `[:foo]`, both interning fresh atoms from input bytes. First-pass `binary_to_existing_atom` round-trip already confirmed a brand-new atom is created. The *sink* is verified; the *untrusted reachability* remains unconfirmed because the CLI caller module is absent from this checkout.
- **Suggested fix** — Keep behaviour (a filter may name a not-yet-interned custom tag) but prefer `String.to_existing_atom/1` for the bare-key branch with a graceful skip on failure, or route keys through an allow-list.

### Ruled out

- **`String.to_integer/1` crash in `parse_kv(:line, line)` (`filters.ex:143-144`)** — `verified`.
  - Axes examined: value-format `{non-integer, empty, negative, zero, > max small-int bignum}` — executed: `parse(["line:notanint"])`→`ArgumentError`, `parse(["line:"])`→`ArgumentError`, `parse(["line:-5"])`→`[line: -5]`, `parse(["line:0"])`→`[line: 0]`, `parse(["line:99999999999999999999999"])`→`[line: 99999999999999999999999]`. Only malformed values crash; negative/zero/huge are accepted but harmless. Reachability: only the developer's own `line:` CLI token.
  - Not examined: none.
  - Verdict: self-inflicted robustness crash of the developer's own `mix test`, no untrusted path. Out of scope.

- **`Enum.EmptyError` / mis-selection in `eval/4` line matching (`filters.ex:259`, `closest_test_before_line/2` @295-303, `describe_block?/2` @289-293)** — `verified`.
  - Axes examined: collection-shape `{empty, full/proper, partial-missing-target}` — executed: `eval([line: 5], [], %{line: 1, describe_line: nil, file: "x"}, [])`→`Enum.EmptyError`; same with a proper `[line 1, line 10]` collection→`:ok`; with `tags.line: 3` (a line matching no test in the collection)→`:ok` (silent mis-select). Reachability: the sole in-tree caller `runner.ex:298` passes the module's **full** `tests` list (read and confirmed at `runner.ex:287-302`); and `runner.ex:311` (`run_module_tests(_, _, _, [])`) short-circuits an empty test list before eval is reached, so the empty-collection crash is unreachable in the shipped path.
  - Not examined: none.
  - Verdict: both behaviours require the caller to violate the documented whole-collection contract, which the only in-tree caller satisfies. Out of scope.

- **`compare/2` "Elixir." prefix stripping (`filters.ex:284-287`)** — `verified`.
  - Axes examined: operand-forms `{atom-vs-atom exact, string-vs-Elixir.-atom, double "Elixir.Elixir." prefix}` — executed: `eval([mod: Foo], …, %{mod: Foo}, [])`→`:ok`; `eval([mod: "Foo"], …, %{mod: Elixir.Foo}, [])`→`:ok`; `eval([mod: "Elixir.Elixir.Foo"], …, %{mod: Foo}, [])`→`:ok`. All match as designed; stripping is bounded (7 chars/recursion) and terminates.
  - Not examined: none.
  - Verdict: affects only which developer-authored tests are selected; crosses no trust boundary. Out of scope.

- **ReDoS via regex `=~` in `has_tag/2` (`filters.ex:267-269`)** — `verified` (source-of-regex).
  - Axes examined: regex provenance — the `%Regex{}` reaches `has_tag/2` only via `eval/4`'s `include`/`exclude`, which `runner.ex:97` derives from `opts[:include]`/`opts[:exclude]` (developer `--only`/name patterns). No in-tree path feeds an externally-sourced regex.
  - Not examined: none within this tree (the CLI/opts origin `mix/tasks/test.ex` is absent, same limitation as the atom finding).
  - Verdict: catastrophic backtracking is self-inflicted; no untrusted-input path in scope.

- **Path handling in `extract_location/1` (`filters.ex:44-60`) and `parse_kv(:location, …)` (`filters.ex:146-147`)** — `verified`.
  - Axes examined: presence of a filesystem sink — read the whole file: `Path.relative_to_cwd`/`String.split`/`Path.split`/`Path.join`/`Integer.parse` are pure string ops; there is **no `File.*` sink in this file**. Executed `parse(["location:test/foo_test.exs:9"])`→`[location: {"test/foo_test.exs", 9}]`; the tuple is returned to Mix as an `:include` opt, not opened here.
  - Not examined: none within this file (traversal, if any, would manifest at a downstream `File.*` in Mix/runner, not here).
  - Verdict: no traversal reachable within scope.

- **`FailuresManifest` deserialization via `failure_info/1` / `fail_all!/1` (`filters.ex:164-177`)** — cross-file; not a defect *in this file*.
  - Axes examined: N/A within this file — `filters.ex:164-176` only forwards the caller-supplied `manifest_file` path to `FailuresManifest.info/1` / `fail_all!/1`. I **read** `failures_manifest.ex` this pass and confirmed the real sink: `safe_binary_to_term/1` (@70-74) calls `:erlang.binary_to_term(binary)` **without the `:safe` option**, rescuing only `ArgumentError`. `info/1` (@55-68) and `read/1` (@45-52) both route through it. So term-decode (atom/ref creation) happens before the outer `{@manifest_vsn, …}` shape match — a genuine deserialization sink — but it lives entirely in `failures_manifest.ex`.
  - Not examined: none relevant to *this* file; the sink module now stands read, so this is verified-by-read rather than assumed.
  - Verdict: cross-file caveat (below), not a finding in `filters.ex`.

## Risks / caveats

- **Unverifiable CLI caller (honest residual):** `parse/1`, `parse_paths/1`, `failure_info/1`, and `fail_all!/1` have **no caller in this checkout**; their documented caller `mix/tasks/test.ex` is absent from the tree, so I could not confirm by reading that only developer-CLI strings reach `String.to_unsafe_atom/1`. The atom-creation *sink* is executed-verified; its *untrusted reachability* is not. The standing Low finding already carries this residual — I did not upgrade it to a higher-confidence finding because the sink is real but self-inflicted under the documented model, and I have no in-tree evidence of an untrusted feed.
- **Cross-file deserialization (now read, still out of scope here):** confirmed by reading `failures_manifest.ex` — `:erlang.binary_to_term` is used without `:safe` (@71). This is the more serious sink but belongs to `failures_manifest.ex`; a follow-up should adjudicate it there, especially the assumption that `_build`/`.mix` manifest files are not attacker-writable.

## Did I verify this?

- **Executed** (Elixir 1.21.0-dev / OTP 29, byte-identical build): atom creation from arbitrary keys; `parse_kv(:line)` across non-int/empty/negative/zero/bignum; `eval/4` across empty/full/partial collections; `compare/2` across atom/string/double-prefix operands; `location:` parsing.
- **Read to verify** (not assumed): `runner.ex:287-311` — the sole in-tree caller passes the full `tests` collection to `eval/4` and short-circuits empty modules; `failures_manifest.ex:45-74` — the `binary_to_term`-without-`:safe` sink and its `ArgumentError`-only rescue.
- **Could not verify:** the CLI caller `mix/tasks/test.ex` (not in this checkout), leaving the untrusted-reachability of `parse/1`'s atom sink asserted, not verified — reflected as `high`-confidence sink / caveated reachability on the finding.

### Dropped in re-review

None. Every first-pass finding and ruled-out entry is preserved; the atom finding is retained, and each ruled-out entry now carries executed-axis evidence with `Not examined: none` (except the ReDoS/atom reachability, which stays caveated because the CLI module is absent from the tree — a limitation stated openly rather than cleared).

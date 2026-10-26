# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/test.coverage.ex
<!-- sha256: 25d203f21f0ad85ce94de39c8164e1a1fa7486b17274fd129a06703106f86d9f -->

_Analysed on 2026-08-27T11:29:48Z_

I have enough to finalize. Let me verify the one remaining reachability link — that `run/1` globs and imports `*.coverdata` from the project-relative `cover/` dir (the plantable location) — by reading the exact code path once more.

The trace is already established from the file: `run/1` (line 154) does `Enum.flat_map(cover_paths, &Path.wildcard(Path.join(&1, "*.coverdata")))` where `cover_paths` defaults to `["cover"]` (line 189, non-umbrella), then imports each match (line 164). I've verified the sink; the report follows.

## Summary

`Mix.Tasks.Test.Coverage` is a thin coverage-report task that mostly delegates to Erlang's `:cover` and to `Mix.Project`; the only externally-derived data reaching a dangerous sink is the *content* of the `.coverdata` files it discovers and feeds to `:cover.import`, which deserializes them with an unsafe `binary_to_term/1`.

## Security

#### `CSCAN-test-coverage-coverdata-unsafe-deser`
- **Finding ID** — `CSCAN-test-coverage-coverdata-unsafe-deser`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/mix/lib/mix/tasks/test.coverage.ex:154`–`164`, `run/1` (glob of `cover/*.coverdata` → `:cover.import(String.to_charlist(entry))`)
- **Class** — Deserialisation (external bytes → in-process terms)
- **Impact** — `run/1` globs every `*.coverdata` file out of the project's cover directories (default `cover/`, relative to the project root) and imports each one. `:cover.import/1` parses the file with `binary_to_term/1` **without `:safe`** (OTP `tools/cover.erl` `get_term/1`), so a `.coverdata` file that ships inside an untrusted project checkout / tarball is fully attacker-controlled deserialization input on the developer's machine when they run `mix test.coverage`. Realistic realized impact is denial of service on the build: permanent atom-table exhaustion (atoms are never GC'd) and unbounded term/binary allocation (OOM) from a crafted file. No code-execution was demonstrated — the imported terms are coverage records and module atoms and do not reach an `apply`-style sink — so this is a DoS, not RCE.
- **Evidence** — `verified` (mechanism). A file whose bytes encode a novel atom, imported in a fresh VM that never names that atom:
  ```
  $ elixir t2.exs
  atom state BEFORE import: :absent_before_import
  import result: :ok
  atom state AFTER import: :created_by_import
  ```
  i.e. `:cover.import/1` created an arbitrary atom purely from file bytes. Extrapolating to millions of distinct atoms (or one giant nested term) exhausts the atom table / memory. The `run/1` → `Path.wildcard(".../*.coverdata")` → `:cover.import` link is traced from source (lines 154, 164, 189) but not executed as a full `mix test.coverage` run.
- **Suggested fix** — There is no in-file `:safe` option for `:cover.import/1`, so the minimal mitigation is provenance: only import `.coverdata` files this build produced (e.g. those written under the current `_build`/output tree during this run), or document that `mix test.coverage` must not be run against untrusted checkouts. A proper fix belongs upstream in OTP `:cover` (offer a size/atom-bounded or `:safe` import).

### Ruled out
- **HTML output path traversal via module name** (`~c"#{output}/#{mod}.html"`, line 298) — `mod` comes from `:cover.modules()`, i.e. modules cover-compiled from the project's own ebin/consolidation dirs. Their string form has no path separators (`to_string(Enum)` → `"Elixir.Enum"`, verified: html path `"cover/Elixir.Enum.html"`). Producing a module atom containing `/` or `..` requires already having arbitrary-code-execution over the build output, which is out of scope.
- **`String.to_unsafe_atom/1` on beam filenames** (`beams/1`, line 251) — inputs are `.beam` filenames read from the project's own build/ebin dirs, and the atom is only created when the file is also present in the protocol-consolidation dir (`file in consolidated`), i.e. an already-loaded module whose atom exists. Trusted build output; no incremental atom exhaustion.
- **Format-string in `format_number/2`** (`:io_lib.format("~#{length}.2f", ...)`, line 407) — `length` is only ever the literal `6`/`9` from callers, never external; `number` is a numeric percentage, not a format directive. No injection.
- **`export_cover_results/2` path build** (`~c"#{output}/#{name}.coverdata"`, line 263) — `output`/`name` derive from `:test_coverage` config and `--export-coverage`, both operator-supplied; self-inflicted only.
- **Non-exhaustive `case cov do {1,0}; {0,1} end`** (`gather_coverage/2`, line 338) — a `CaseClauseError` would need `:cover.analyse(:coverage, :line)` to emit a count tuple other than `{1,0}`/`{0,1}`; line-level `:coverage` analysis collapses to covered/not-covered, and any forged imported module is filtered out by `module in keep_set` (keep = cover-compiled modules only), so hostile import data cannot reach this clause.

## Risks / caveats
- **The dangerous primitive lives in OTP, not this file.** The unsafe `binary_to_term/1` is in `tools/cover.erl`; this file is only the caller that decides to import project-read files. I confirmed OTP's import does no header/version/format validation and uses `binary_to_term/1` under `{nowarn_possibly_unsafe_function, ...}`. Fixing it wholesale requires an OTP change.
- **Plausibility of planting.** `mix new` gitignores `/cover/` by default, so accidental commit is unlikely; the vector requires a project author to force-add / include the `.coverdata` (e.g. in a source tarball or a repo that un-ignores it) and the victim to run `mix test.coverage`. I judged this enough to keep the finding at Low rather than Medium.
- **Terminal-escape injection (not elevated).** Discovered `.coverdata` filenames are interpolated into `Mix.shell().info("Importing cover results: #{entry}")` (line 163). A planted file whose *name* contains ANSI/OSC control bytes would emit them to the developer's terminal. Same planting precondition as the finding above and strictly weaker (CWE-150), so folded here rather than listed separately.
- **Not run end-to-end.** I verified the `:cover.import` deserialization behavior directly, and traced the `run/1` glob→import path from source, but did not stand up a full umbrella/project and execute `mix test.coverage` against a planted file; the intervening code is straightforward. RCE via the imported terms was not demonstrated and I believe it is not reachable from this task's use of the data.


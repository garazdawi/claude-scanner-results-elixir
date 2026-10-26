# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/filters.ex
<!-- sha256: d4582490adf5fbfc759cd0ee15d6ffa04c99ef7be16f95c7875ec167a5405213 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:34:20Z_

I have enough verified evidence. Let me write the report.

## Summary

Two contract-level defects surfaced in `filters.ex`: the `@spec`/`@type` for `parse_paths/1` understates the shape actually returned, and `parse/1` crashes with an opaque error on a non-integer `line:` filter that the path-parsing code path handles gracefully.

## Bugfixes

#### `CSCAN-filters-parse-paths-location-type-too-narrow`
- **Finding ID:** `CSCAN-filters-parse-paths-location-type-too-narrow`
- **Severity:** Low
- **Confidence:** high
- **Location:** `filters.ex:11-13` (`@type location`, `@type ex_unit_opts`), consumed by `@spec parse_paths/1` at line 30 and the deprecated `parse_path/1` at line 18.
- **Class:** Contract violation (`@spec`/`@type` disagrees with implementation).
- **Impact:** Dialyzer analysis of any caller that pattern-matches or forwards the `include:` list produced by `parse_paths/1`/`parse_path/1` gets a wrong contract: the declared `include` element type is only `{:location, {String.t(), pos_integer | [pos_integer, ...]}}`, but when *any* path carries a line number, the result also contains bare-binary entries `{:location, String.t()}` for the paths that do **not** carry one. Downstream code that trusts the spec (e.g. a `case`/match assuming the tuple form) is analyzed against an incomplete type, propagating false negatives/positives through Dialyzer.
- **Evidence:** `verified` —
  ```
  $ elixir -e 'IO.inspect ExUnit.Filters.parse_paths(["a.exs", "b.exs:12"])'
  {["a.exs", "b.exs"],
   [exclude: [:test], include: [location: "a.exs", location: {"b.exs", 12}]]}
  ```
  `location: "a.exs"` (binary second element) is not admitted by `@type location`, yet `has_tag({:location, path}, %{file: file}, _) when is_binary(path)` (line 239) is written precisely to consume it, so the value is intended, and the type is the thing that is wrong.
- **Suggested fix:** Widen the type to include the bare-path form, e.g. `@type location :: {:location, String.t() | {String.t(), pos_integer | [pos_integer, ...]}}` (or split into two `:location` variants), so `ex_unit_opts` reflects the mixed `include` list actually returned.

#### `CSCAN-filters-parse-line-noninteger-crash`
- **Finding ID:** `CSCAN-filters-parse-line-noninteger-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `filters.ex:143-144`, `parse_kv(:line, line)` (reached from public `parse/1`, line 134).
- **Class:** Error-handling asymmetry / non-total function on documented input.
- **Impact:** `parse/1` is documented as parsing filters "as one would receive from the command line" (`mix test --only line:...` / `--exclude`). A non-integer value such as `line:abc` raises a raw `ArgumentError` from `:erlang.binary_to_integer/1` mid-`Enum.map`, aborting the whole filter parse with a stack trace that never mentions the offending filter. The sibling code path for line numbers embedded in file paths (`validate_line_number/1`, line 69-73) deliberately degrades gracefully with an `IO.warn`, so the crash is an inconsistency, not a designed contract.
- **Evidence:** `verified` —
  ```
  $ elixir -e 'IO.inspect ExUnit.Filters.parse(["line:abc"])'
  ** (ArgumentError) ...
      :erlang.binary_to_integer("abc")
      lib/ex_unit/filters.ex:144: ExUnit.Filters.parse_kv/2
      (elixir) lib/enum.ex:1725: Enum."-map/2-lists^map/1-1-"/2
  ```
- **Suggested fix:** Guard the conversion, e.g. use `Integer.parse/1` and either fall through to `parse_kv(key, value)` (treat as an opaque tag) or emit the same `IO.warn` used for invalid line numbers, rather than letting `String.to_integer/1` raise an opaque error.

## Risks / caveats
- The `parse/1` line-crash is borderline: raising on malformed input can be an acceptable contract, and the `ArgumentError` text ("not a textual representation of an integer") is partly informative. I rate it Low and flag it primarily because a graceful sibling path (`validate_line_number/1`) exists; a maintainer may reasonably deem current behavior intended.
- I verified `eval/4` line/location matching end-to-end against constructed `ExUnit.Test` collections (the `:infinity` sentinel in `closest_test_before_line/2` relies on Erlang term ordering placing numbers below atoms — confirmed correct), and ran all `normalize/2`, `parse/1`, and `eval/4` doctests: all pass.
- I did not exercise `failure_info/1`/`fail_all!/1`, which are thin delegations to `ExUnit.FailuresManifest` (out of this file's scope).

### Ruled out
- **`closest_test_before_line/2` `:infinity` sentinel / `Enum.min_by` ties** — verified `line:5..30` against a 3-test collection returns the correct closest-before test each time (output above); numbers sort below the `:infinity` atom, so "no test before line" is handled. Not a bug.
- **`closest_test_before_line/2` on empty collection raising `Enum.EmptyError`** — reachable only if `eval/4` is invoked with a `:line`/`:location` filter and an empty `collection`, but the matching `has_tag/3` clauses require `tags` carrying `:line`/`:describe_line`, i.e. a real test whose own module supplies a non-empty collection; I could not construct a documented call that reaches it with `[]`, so it stays out of scope as internal-misuse-only.
- **`eval/4` returning `:ok` when an `include` filter does not match** — verified this is by design: without an `exclude` filter the third `cond` branch (`true -> maybe_skipped`) returns `:ok`; `parse_paths` always pairs line/location includes with `exclude: [:test]`, and with that pairing selection is correct (verified). Not a bug.
- **`eval/4` exclude message `"due to true filter"`** for location/line excludes (`has_tag/3` returns a boolean, not the key, at lines 240/246/249) — cosmetic only, and excluding by `:location`/`:line` is not a documented/normal usage; no wrong selection results. Not reported.
- **`normalize/2` precedence logic** — all eight published doctests reproduced exactly (output above); the `exclude_included` computation drops the right keys. Not a bug.
- **`extract_location/1` on `line:0`/negative/Windows-drive paths** — traced and confirmed these fall through to the "no line" branch (`{path, path}`) without crashing; graceful, no finding.


# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/failures_manifest.ex
<!-- sha256: 4436ac8ec8dd950597701bbab59c4611ca7446b5e0263e339f7f86fc3618a6f5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:31:03Z_

## Summary

One real defect: `info/1` crashes with `CaseClauseError` on a decodable-but-malformed manifest payload, while its sibling `read/1` degrades gracefully on the same input — an asymmetry that violates the folder's "corruption tolerated" contract.

## Bugfixes

#### `CSCAN-failures_manifest-info-caseclause-noncatchall`
- **Finding ID:** `CSCAN-failures_manifest-info-caseclause-noncatchall`
- **Severity:** Low
- **Confidence:** high
- **Location:** `failures_manifest.ex:55-68`, `info/1` (the inner `case manifest do ... end`)
- **Class:** Missing clause / non-total function (`case` without catch-all)
- **Impact:** A manifest file whose contents decode (via `binary_to_term`) to `{1, X}` where `X` is neither a map nor `:all` makes `info/1` raise `CaseClauseError` instead of returning the documented empty result. `info/1` is the entry point ExUnit calls at suite start to decide which files/tests to rerun; a crash here aborts the run before any test executes. The sibling `read/1` reading the exact same bytes returns `%{}` cleanly, which shows the intended behaviour is graceful degradation and that `info/1` is the one that's wrong. The `@spec` promises `{MapSet, MapSet} | :all`, never a raise.
- **Evidence:** `verified`.
  ```
  $ elixir t.exs   # requires the file, writes crafted manifests, calls info/read
  info {1,:garbage}: {:raised, CaseClauseError}
  read {1,:garbage}: %{}
  info :all: :all
  read :all: %{}
  info {1,42}: {:raised, CaseClauseError}
  ```
  The crafted file is `:erlang.term_to_binary({1, :garbage})` — a valid `@manifest_vsn`-1 tuple with a non-map payload. `binary_to_term` succeeds (so `safe_binary_to_term` returns `{:ok, ...}`), the `with` matches `{@manifest_vsn, manifest}` with `manifest = :garbage`, and the `case` has no catch-all → `CaseClauseError`. `read/1` avoids this because its `with` pattern constrains the payload with `%{} = manifest`, so a non-map payload fails the match and falls to `else -> new()`.
- **Suggested fix:** Give the `case` a catch-all that mirrors `read/1`'s graceful default, e.g. add a final clause `_ -> {MapSet.new(), MapSet.new()}` after the `%{}` clause (or constrain the `with` pattern to `{@manifest_vsn, manifest} when is_map(manifest) or manifest == :all`). Smallest change:
  ```elixir
  case manifest do
    :all -> :all
    %{} -> {manifest |> Map.values() |> MapSet.new(), manifest |> Map.keys() |> MapSet.new()}
    _ -> {MapSet.new(), MapSet.new()}
  end
  ```

## Risks / caveats

- Reachability of the one finding is bounded: natural truncation of a `term_to_binary`-encoded `{1, %{...}}` almost always fails `binary_to_term` (raising `ArgumentError`, which `safe_binary_to_term` already converts to `:error` → graceful). Hitting the `CaseClauseError` requires a fully-valid encoded term of shape `{1, non_map_non_all}`, i.e. a deliberately crafted or externally-tampered manifest, hence Low severity. It is nonetheless a genuine contract gap (the module's own `read/1` handles the case) and a one-line fix.
- I could not exercise `info/1`/`read/1` inside a full `mix test --failed` run in this sandbox; the verification loaded the module standalone and drove the public functions directly, which is sufficient for the decode/`case` path but does not exercise integration with the runner.

### Ruled out
- **`read/1` returns `%{}` for an `:all` manifest instead of `:all`** — Not a bug. `read/1`'s `@spec` is `read(Path.t()) :: t` and `t` is a map type that does not include `:all`; the `%{} = manifest` guard is deliberate, and the runner uses `info/1` (which does return `:all`) for the fail-all decision. Verified `read` returns `%{}` for an `:all` file (output above), consistent with the spec.
- **`put_test/2` has no head for a `{:passed, _}` state** — Not reachable. Grepped `runner.ex`: passing tests keep `state: nil` (only `:invalid/:failed/:skipped/:excluded` states are ever assigned), and the `state: nil` head handles them (removing the now-passing test from the manifest). No `{:passed, _}` shape is ever produced.
- **`@manifest_vsn` mismatch (version bump) crashing `read`/`info`** — Handled. The `with` pattern `{@manifest_vsn, ...}` fails to match a different version tuple and both functions fall to their `else`/graceful branch. Verified conceptually via the same decode path; an old-version tuple simply doesn't match the pinned version.
- **`find_deleted_tests/3` infinite loop on the `file_exists == :error` retry** — Ruled out. The `:error` branch inserts the key into `file_existence` before recursing on the same list, so the subsequent `Map.fetch` returns `{:ok, _}` and cannot re-enter the `:error` branch; the recursion is finite. Reasoned from the code; the branch is a one-shot cache-fill.
- **`safe_binary_to_term/1` only rescuing `ArgumentError`** — Correct as written; `:erlang.binary_to_term/1` raises `ArgumentError` for malformed/truncated binaries, which is precisely what is rescued. No other exception class escapes on the tested inputs.


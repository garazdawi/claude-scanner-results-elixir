# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/cover.exs
<!-- sha256: 410fbc7779d6b2b2866d9015acd8ffa3cce7413f89c7a21c6f3e034ba73d2de9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:02:07Z_

# Coverage-combine script review — `lib/elixir/scripts/cover.exs`

## Summary
The script combines and reports test coverage correctly on the normal path, but exports the combined coverdata without first ensuring the target directory exists, so it aborts with an opaque `MatchError` whenever the `cover/` directory is absent.

## Bugfixes

#### `CSCAN-cover-export-missing-mkdir`
- **Finding ID** — `CSCAN-cover-export-missing-mkdir`
- **Severity** — Low
- **Confidence** — high
- **Location** — `cover.exs:18`, top-level script body (`:ok = :cover.export(String.to_charlist(coverdata_output))`)
- **Issue** — `:cover.export/1` writes `combined.coverdata` into `CoverageRecorder.cover_dir()`, but nothing creates that directory first. When the directory does not exist, `:cover.export` returns `{:error, {:cant_open_file, path, :enoent}}`, which fails the `:ok = ...` match and raises `MatchError`. Every other export site in the tree guards with `File.mkdir_p!` beforehand — `cover_record.exs:45` (`File.mkdir_p!(cover_dir())` right before its `:cover.export` at line 47) and the Mix task's `export_cover_results`/`html` (`File.mkdir_p!(output)` at `test.coverage.ex:261` and `:294`) — so this line is the lone unguarded export.
- **Class** — Missing clause / non-total function (unhandled `{:error, _}` from `:cover.export`, crashing the `:ok =` match).
- **Impact** — Running the combine script before any coverage has been recorded (so `cover/` was never created) aborts with a bare `MatchError` at `cover.exs:18` rather than a clear "no coverage data" message or a created directory. It's a build/CI convenience script and the directory usually pre-exists from a prior `COVER=1` test run, so the blast radius is small — a confusing failure, not data loss.
- **Evidence** — `verified`. Reconstructed the directory layout in `/tmp/repo` (so `root_dir()` resolves correctly) with one `ex_unit_demo.coverdata` input and a cover-compilable beam.
  - With `cover/` present: `elixir cover.exs` prints the summary table, writes `combined.coverdata` + HTML, exits 0.
  - After `rm -rf /tmp/repo/cover`, `elixir cover.exs` produces:
    ```
    ** (MatchError) no match of right hand side value:
        {:error,
         {:cant_open_file,
          ~c".../cover/combined.coverdata", :enoent}}
        cover.exs:18: (file)
    ```
    (exit 1). Re-creating the empty directory makes it succeed again (reports 0%, exit 0), confirming the missing-directory case is the sole trigger.
- **Suggested fix** — Insert `File.mkdir_p!(CoverageRecorder.cover_dir())` immediately before line 18 (mirroring `cover_record.exs:45`), or match/handle the `{:error, reason}` return with an actionable message.

## Risks / caveats
- The `cover_dir()` path in this sandbox resolves through the repo-root symlink to `/cover`, which is unwritable; I reproduced both the happy path and the crash in a faithful `/tmp/repo/lib/elixir/scripts` copy so `root_dir() = __DIR__/../../..` resolved to a writable tree. The behavior is layout-independent (missing target dir → `enoent` → `MatchError`).
- I did not exercise the `enable_coverage()` path against the real Elixir `lib/*/ebin` beams (the scanned copy ships no `.beam` files); I substituted one cover-compilable beam. The tokenizer/Unicode conformance material in the standing threat-model context does not apply to this file — it is a coverage build script with no identifier, parser, or untrusted-input surface.

### Ruled out
- **Group-leader redirect (`cover.exs:24`) fails to silence cover's analyse output** — Not a bug. `cover_pid` is the cover-server pid returned by `:cover.start()` (via `enable_coverage/0`); redirecting its group leader silences the server's internal `io` during `generate_cover_results`, exactly as the upstream Mix task does at `test.coverage.ex:169`. The main-process `Mix.shell()` summary/HTML messages still print (verified: table and "Generated HTML..." appeared on stdout). Basis: `verified`.
- **`Application.ensure_all_started(:mix)` ordering / `Mix.shell()` unavailable in `generate_cover_results`** — Not a bug. `:mix` is started (line 20) before the redirect and the report call; the full run completed with the summary table and HTML generation. Basis: `verified`.
- **Combined output re-imported / double-counted on re-run** — Not a bug. The input wildcard is `ex_unit_*.coverdata` (`cover.exs:10`) and the output is `combined.coverdata` (line 12), which never matches; a second run produced the same 50%/0% figures rather than doubling. Basis: `verified` (ran twice with `cover/` populated).
- **Empty input set produces a crash** — Not a bug when the directory exists: with `cover/` present but no `ex_unit_*` files, the loop is a no-op, `:cover.export` writes an empty combined file, and the report shows 0% (exit 0). The crash occurs only in the distinct missing-directory case reported above. Basis: `verified`.
- **Relative shebang `#!bin/elixir` (line 1)** — Not a functional defect on the intended invocation (`bin/elixir lib/elixir/scripts/cover.exs` from repo root, where the shebang is unused). It would only bite someone who `chmod +x`'d the file and executed it as `./cover.exs` from a non-root cwd; that is outside the documented run path. Basis: `asserted` (invocation site lives in the repo Makefile/CI, which is not part of the scanned `lib` subtree, so I could not read it directly).


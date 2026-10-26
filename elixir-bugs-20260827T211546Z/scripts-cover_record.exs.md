# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/cover_record.exs
<!-- sha256: a9d32196f9f64b19aed90c3be792d3ea65abf11251c9fe9f15f4da96d843db62 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:03:52Z_

## Summary
`cover_record.exs` is a small, standalone coverage-recording build script whose documented surface (`maybe_record/1`, `enable_coverage/0`, `cover_dir/0`) behaves correctly under testing, and I found no reachable bugs.

## Bugfixes

No findings

### Ruled out

- **`String.to_unsafe_atom/1` doesn't exist / is deprecated (line 88)** — Rejected. In this tree (Elixir 1.21.0-dev) `String.to_unsafe_atom/1` is the *current* API; it is `String.to_atom/1` that carries `@doc deprecated: "Use to_existing_atom/1 or to_unsafe_atom/1 instead"` (`lib/elixir/lib/string.ex:3010,3039`). The script uses the non-deprecated function. Basis: `verified` (grep of `string.ex`).
- **`:docs_v1` tuple match on line 89 has wrong arity / metadata in wrong position** — Rejected. `Code.fetch_docs/1` returns a 7-tuple `{:docs_v1, anno, lang, format, module_doc, metadata, docs}`; the pattern `{:docs_v1,_,_,_,_,%{deprecated: _},_}` places `%{deprecated: _}` at position 6 (metadata), which is correct, and deprecated modules do carry `:deprecated` there. Basis: `verified` — `Code.fetch_docs(Enum)` → `{:docs_v1, 7}`; `Code.fetch_docs(HashDict)` metadata contains `deprecated: "Use Map instead"`.
- **`skip_from_coverage?/1` crashes on doc-less / Erlang / protocol beams** — Rejected. Non-`:docs_v1` returns (`{:error, :module_not_found}`, `{:error, :chunk_not_found}`) simply fail `match?/2` and yield `false`; the `mod in @to_skip` `or` short-circuits before `fetch_docs` for skip-listed atoms. Basis: `verified` — ran the exact predicate over all 447 installed `lib/*/ebin/*.beam` files: 0 raised/caught, 20 correctly skipped.
- **`:cover.compile_beam/1` given a list is a contract violation / wrong return shape** — Rejected. `cover:compile_beam/1` explicitly accepts `[beam_mod_file()]` and returns `[compile_beam_result()]` (`tools-4.2.1/src/cover.erl:513-527`), so mapping to a charlist list and `Enum.each` over `{:ok,_} | {:error,_}` results is correct; an empty list falls through to the list clause and returns `[]`. Basis: `verified` — read cover.erl spec/clauses; end-to-end `maybe_record("smoke")` cover-compiled the ebins without raising.
- **`maybe_record/1` end-to-end crash** — Rejected. With `COVER=true`, `maybe_record/1` returned `true` and completed `enable_coverage` (stop/start/cover-compile) with no error; the only failure observed was `File.mkdir_p!` on the sandbox's read-only tree inside the `at_exit` callback, which is an environment constraint, not a code defect. Basis: `verified` (ran it).
- **Path traversal via `suite_name` interpolated into the filename (line 40)** — Rejected as out of scope. `suite_name` originates from build/test configuration, not from a documented untrusted-input entry point, so it does not meet the "untrusted data through a public path" bar. Basis: `asserted` (no untrusted caller exists in this snapshot).

## Risks / caveats
- **Multi-suite reuse within one VM (not a confirmed bug).** If `maybe_record/1` were called more than once in a single VM, each `enable_coverage/0` performs `:cover.stop()`/`:cover.start()`, discarding previously accumulated coverage, while every registered `System.at_exit` still runs — so earlier `.coverdata` files would capture only the surviving (later) run's data. I could not confirm this matters: no caller of `CoverageRecorder`/`maybe_record` exists anywhere in this repo snapshot (searched the whole tree), and Elixir's normal usage is one suite per VM. If a caller ever records multiple suites in one VM, revisit this.
- I could not exercise the `at_exit`/`:cover.export` path to completion because the sandbox filesystem is read-only; that branch was validated only up to the `mkdir_p!` call.


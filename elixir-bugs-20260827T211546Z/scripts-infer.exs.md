# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/infer.exs
<!-- sha256: ef90989f937ef462551197313e25aa898bcf06ff711ad609ac3e334bdde65054 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:05:05Z_

# Review: `infer.exs`

- **Summary:** This file is a build-time compilation script (`.exs`) with no exported/public functions, no documented API surface, and no untrusted input — it runs once against Elixir's own freshly-compiled `ebin/*.beam` files during the stdlib build, so nothing here is reachable through a documented public API.

## Bugfixes

No findings

The file is a top-level script executed by the build system (`elixir infer.exs`). It defines no module, no public/exported function, no behaviour callback, and processes no attacker- or caller-supplied data — every input (`Path.expand("../ebin")`, `Application.spec(:elixir, :modules)`, the `.beam` binaries it reads) is produced by the trusted build of Elixir itself. Under the stated scope ("only report bugs reachable through the file's documented public API … ignore findings only reachable through misuse of internal interfaces"), there is no public path into this code, so no in-scope bug class applies.

### Ruled out

- **Line 75 `Enum.map(fn {:ok, {module, path, new_binary}} -> …)` does not handle `{:exit, reason}` from `Task.async_stream`.** A crashing/exiting task would `MatchError` here rather than being handled. Ruled out as an in-scope finding: this is a fail-loud build script, the stream runs over trusted first-party modules with `timeout: :infinity` (no timeout-exit shape), and there is no public API or untrusted input that could reach this path — a hard match failure that aborts the build is the intended contract for a compiler bootstrap step. Basis: asserted (reasoned from `Task.async_stream` semantics; the script cannot be run standalone without a full in-progress stdlib build).
- **Line 84/93 `div(time, 1000)` for millisecond reporting.** `:timer.tc/1` returns microseconds, so `div(time, 1000)` correctly yields milliseconds; the label "ms" matches. Not a bug. Basis: asserted (documented `:timer.tc` unit is microseconds).
- **Lines 10, 12, 20 assertion matches (`[:elixir] = …`, `[_ | _] = …`, `true = …`).** These are deliberate build-time sanity assertions designed to abort the build if the environment is wrong; a match failure is the intended behaviour, not a defect. Basis: asserted (comments at lines 9, 19 state the intent).
- **Line 60 string `"ExCk"` vs line 65 charlist `~c"ExCk"`.** `:elixir_erl.checker_chunk/2` returns the chunk id as the binary `"ExCk"` while `List.keyreplace` targets the charlist `~c"ExCk"` used by `:beam_lib.all_chunks/1`; these operate on two different lists (the freshly built chunk list vs. the original `chunks`) and the mismatch is by construction, not an error. Basis: asserted (reading of the two distinct data flows).

## Risks / caveats

- I could not execute this script in isolation: it requires a fully built `../ebin` with `debug_info` and `ExCk` chunks plus internal modules (`Module.ParallelChecker`, `Module.Types.infer/7`), so all reasoning above is `asserted` rather than `verified`. The ruled-out `{:exit, _}` match-gap is real code behaviour but falls outside the "documented public API" scope; if the scope were widened to build-robustness it would merit a Low-severity note (a single module crash aborts the whole inference pass with an opaque `MatchError` instead of a message naming the failing module).


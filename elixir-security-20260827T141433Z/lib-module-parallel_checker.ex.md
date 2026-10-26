# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/parallel_checker.ex
<!-- sha256: bbb5ae467747a4f014f29d7e546d02323ccd8d5d982bf2d8915618ee2586f713 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:52:10Z_

All claims verified. Writing the report.

---

## Summary

`Module.ParallelChecker` deserializes and dispatches on BEAM chunk data (`ExCk`, `debug_info`) reached through its documented API (`verify/2`, `fetch_export/5`, `spawn/6`) using non-`:safe` `binary_to_term` and `with/else` chains that catch pattern failures but **not exceptions**, so a crafted/corrupt BEAM crashes or hangs the checker — and, via the process link it holds to the compiler, the whole compilation.

## Security

#### `CSCAN-parallel_checker-beam-chunk-crash-dos`
- **Finding ID:** `CSCAN-parallel_checker-beam-chunk-crash-dos`
- **Severity:** Medium
- **Confidence:** medium
- **Location:** `parallel_checker.ex:118-134` (`spawn_and_register_cache/6`), `parallel_checker.ex:431-436` (`cache_module/2`); crash-propagation at `:89`/`:110`; unbounded wait at `:199`/`:209-228`
- **Class:** Resource consumption (DoS) / Deserialisation
- **Issue:** Both BEAM-parsing `with` chains route failures to `else _ ->`, but that only handles *non-matching return values* (e.g. `File.read` → `{:error,_}`, `:beam_lib.chunks` → `{:error,_}`, version-tuple mismatch). It does **not** handle *exceptions*, and three RHS expressions in the success path raise on hostile input: `:erlang.binary_to_term/1` (line 122/433) raises `ArgumentError` on malformed `ExCk` bytes; `backend.debug_info(...)` (line 121) raises `UndefinedFunctionError` when the file-supplied `backend` atom has no `debug_info/4`; and `cache_chunk/3` (`contents.exports`, line 501) / `module_map_to_module_tuple` raise `BadMapError`/`KeyError` when a *valid* term decodes to the wrong shape. Any of these kills the worker. Because the cache worker does `Process.link(pid)` (line 110) and the checker worker does `Process.link(pid)` (line 89) to the compiler *before* parsing, the crash propagates through the link and takes the compiler down; if the compiler traps exits, the `{:EXIT,…}` is unmatched by `collect_results/3` (line 213-228, a bare `receive` with no `after`) and the `:infinity` waits in `verify/2` hang forever instead.
- **Impact / threat:** An attacker who can get a crafted `.beam` onto the compile-time code path (a precompiled/`:path` dependency, a cached artifact) or who is passed one via the documented `verify/2`/`fetch_export/5` APIs can abort or wedge the build. The precondition is only that the target module is *referenced* (so `fetch_export`→`cache_module` reads its object code) or handed to `verify/2` — not that its code executes. Trust boundary crossed: untrusted compiled artifact → build process.
- **Evidence:** `verified`.
  - `:erlang.binary_to_term(<<131,255,255,255>>)` → `{:raised, ArgumentError}`; wrapping it in `with {1,c} <- … do … else _ -> :caught end` → `{:escaped_with_as_exception, ArgumentError}` (the `else` never fires).
  - Faithful mini-repro of the cache worker (monitor parent → `:cache` → `Process.link(parent)` → raising `binary_to_term`): parent process was killed by the propagated link exit — `** (EXIT from #PID<…>)`, and the parent's post-receive `IO.inspect` lines never ran.
  - `cache_module` path: malformed bytes → `{:uncaught, ArgumentError}`; valid ETF `{:elixir_checker_v10, [:not,:a,:map]}` (version tuple matches, `:elixir_checker_v10` already interned) → `{:uncaught, BadMapError}` — neither reaches `else`.
- **Suggested fix:** Treat chunk decoding as fallible: wrap `binary_to_term`, `backend.debug_info`, and the shape-dependent `cache_chunk`/`module_map_to_module_tuple` in `try/rescue` (or a helper returning `{:ok,_}|:error`) so failures fall through to the existing `:uncached`/fallback branch instead of raising. Additionally, avoid escalating a worker failure into a compiler kill/hang: give `collect_results/3` a way to observe worker death (monitor `DOWN` / handle `{:EXIT,…}`) so a bad artifact yields a diagnostic, not an unbounded `:infinity` wait.

#### `CSCAN-parallel_checker-unsafe-binary-to-term-atoms`
- **Finding ID:** `CSCAN-parallel_checker-unsafe-binary-to-term-atoms`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `parallel_checker.ex:122` (`spawn_and_register_cache/6`) and `parallel_checker.ex:433` (`cache_module/2`)
- **Class:** Deserialisation (CWE-502) / Atom creation (CWE-770)
- **Issue:** `:erlang.binary_to_term/1` is called without `[:safe]` on the raw `ExCk` chunk bytes of a BEAM file. Decoding runs to completion — materialising every atom/fun/ref in the term — *before* the `{@elixir_checker_version, contents}` pattern is tested, so a term full of unique atoms exhausts the (never-GC'd) atom table even though it will ultimately be rejected.
- **Impact / threat:** A crafted BEAM on the compile-time code path (see reachability above) can permanently consume the atom table and abort the VM with `system_limit`, a DoS of the build. This is a resource-exhaustion escalation of the same untrusted-artifact boundary; it does not add code execution (the raw bytecode already would, if loaded).
- **Evidence:** `verified`. A hand-built `ATOM_EXT` for a never-before-seen name decoded via non-`:safe` `binary_to_term` interned the atom (`String.to_existing_atom` went `false → true`, value `:cscan_atom_1348`); the identical bytes under `[:safe]` returned `{:rejected, ArgumentError}`.
- **Suggested fix:** Bound the risk before decoding (reject oversized `ExCk` chunks) and prefer `[:safe]` where feasible. Note the tension: `ExCk` for a not-yet-loaded module may legitimately contain new atoms, so `:safe` is not a clean drop-in — a size cap plus the `try/rescue` from the previous finding is the pragmatic mitigation.

#### `CSCAN-parallel_checker-beam-metadata-code-exec`
- **Finding ID:** `CSCAN-parallel_checker-beam-metadata-code-exec`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `parallel_checker.ex:121` (`backend.debug_info/4` dispatch) and `parallel_checker.ex:292-294` (`apply(verify_mod, verify_fun, [module])`)
- **Class:** Protocol and behaviour dispatch / Code execution
- **Issue:** Two values decoded *from the BEAM file itself* drive dynamic invocation. `backend` in `{:debug_info_v1, backend, data}` is used as `backend.debug_info(:elixir_v1, …)`, so a file-chosen module atom is dispatched to (and code-loaded, running any `@on_load`). The resulting module map's `after_verify` list is later invoked as `apply(verify_mod, verify_fun, [module])` in `check_module/1` — an arbitrary MFA taken from file metadata, executed on the build machine when the module is fully checked (the `verify/2` runtime-files path and the protocol-consolidation re-read path).
- **Impact / threat:** Passing an untrusted precompiled BEAM to `verify/2` (or having one consolidated) executes decoder/`after_verify` code that the operator did not write — a "type-check, don't run" expectation is violated. Realistically this coincides with the inherent danger of processing attacker-supplied compiled artifacts (and mirrors OTP `beam_lib`'s own `backend.debug_info` mechanism and the intended `@after_verify` feature), which is why it is rated Low rather than higher.
- **Evidence:** `asserted`. The dispatch/`apply` on file-derived atoms is direct in the source; I did not build an end-to-end malicious `.beam` to observe the callback firing.
- **Suggested fix:** If untrusted BEAMs are ever in `verify/2`'s scope, allow-list `backend` (e.g. only `:elixir_erl`) before dispatch and treat `after_verify` MFAs as untrusted (skip, or restrict to the module under check). At minimum, document that `verify/2`/`fetch_export/5` must only be pointed at trusted artifacts.

### Ruled out
- **`Path.absname/1` (lines 217, 391) and `String.to_charlist(Path.relative_to_cwd/1)` (line 421).** Used only to normalise paths for diagnostic display/stacktraces; no `File` read/write/delete is performed on them, so no traversal sink. Basis: traced every use of the resulting path — all terminate in a diagnostic map or stacktrace tuple.
- **Public `:ets` table (line 547, `:public`).** Any co-tenant BEAM process could write it, but the `tid` is not published and the threat model explicitly excludes a co-tenant process as a boundary the compiler defends. Basis: table id only escapes inside the `cache()` handle returned to the caller.
- **`String.to_atom`/`Module.concat`/`Code.eval*`/`System.cmd`/shell/`Regex` from interpolation.** None present in this file. Basis: read the whole file; the only atom-creating primitive is `binary_to_term` (covered above), and no code-eval/command/regex sink exists.
- **pdict handling in `verify/1` and `get/0` (lines 46-56, 151-182).** Save/restore of `:elixir_checker_info`/`:elixir_compiler_info` is process-local; the check-then-start in `get/0` is not cross-process. No externally-derived key/value. Basis: values are the caller's own `self()`/cache handle, never attacker-derived.

## Risks / caveats

- **Reachability of crafted BEAM bytes.** The strongest exposure for findings 1–2 assumes attacker-controlled *bytes* in an `ExCk`/`debug_info` chunk. In the common Hex flow, dependencies ship source and are recompiled locally, so their chunks are emitted by the local (trusted) compiler; the crafted-bytes vector therefore needs a precompiled/`:path` dependency, a cached/fetched artifact, or a direct `verify/2`/`fetch_export/5` caller pointed at an untrusted file. I verified the crash/atom mechanics and the link propagation, but did **not** stand up a full `mix compile` against a malicious dependency, so the end-to-end supply-chain reproduction is asserted rather than run.
- **Compiler exit-trapping.** Whether a crash *kills* the compiler or *hangs* it (finding 1) depends on whether the enclosing compiler process traps exits; both outcomes are DoS. I confirmed the non-trapping (kill) case directly and reasoned the trapping (hang) case from `collect_results/3` lacking an `{:EXIT,…}` clause and any `after`.
- **Finding 3 is partly inherent.** Executing `after_verify`/loading `backend` when reading a BEAM overlaps with the unavoidable risk of processing untrusted compiled artifacts and with intended language features; it is reported for completeness at Low, and its callback-firing step is asserted, not executed.
- This module is `@moduledoc false`; I treated the `@doc`-annotated functions (`start_link/1`, `stop/1`, `get/0`, `put/2`, `spawn/6`, `verify/1,2`, `fetch_export/5`) as the documented surface. Findings driven purely by malformed *internal* messages to the gen_server or by violating the `@spec` contracts were excluded per scope.


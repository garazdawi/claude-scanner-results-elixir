# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_code_server.erl
<!-- sha256: ecdc81913d4a34bb93a93d1c450c11b2b9aa0ec94d0059a4363566ae76bc179c -->

_Analysed on 2026-08-27T14:39:02Z_

I have enough to finalize. I traced every sink to its origin: all attacker-influenceable values (module names, paths, pids, tuples) either arrive already-atomised from upstream, are used only as inert map keys, or originate from internal compiler callers with literal message tags.

# Security review: `elixir_code_server.erl`

## Summary
`elixir_code_server` is a singleton `gen_server` serialising module definition, the required-files queue, and the compiler-module pool; every security-relevant sink is driven by internal compiler callers (`elixir_compiler`, `elixir_module`) with programmer-constructed message tags, and no attacker-controlled *content* from untrusted source code reaches a dangerous operation in this file, so there are no in-scope findings.

## Security
No findings.

## Risks / caveats
- **Public ETS table (`elixir_modules`, line 31).** The module registry is `public`, so any co-tenant BEAM process can insert bogus module tuples or delete legitimate entries, corrupting the `defmodule` duplicate-detection invariant. This is out of scope per the folder threat model ("A co-tenant BEAM process is not a boundary the compiler defends") and is not reachable with attacker-controlled *content* through any documented public API — untrusted source code never gets to choose ETS operations.
- **Unknown-message crash (`{stop, {badcall,…}}` line 81, `{stop, {badcast,…}}` line 127).** An unexpected message takes the singleton down, aborting all in-flight compilation and dropping queued `acquire` waiters. This is a deliberate fail-fast design and is only reachable by sending crafted messages to an internal, undocumented process — explicitly out of scope.
- **Cross-file atom-pool dependency (`compiler_module/1`, line 161).** The `elixir_compiler_N` counter is monotonic and only bounds atom growth *if* every retrieved module is returned. Retrieval (`elixir_compiler.erl:82`) happens *before* the `try/after` in `dispatch/4` (`elixir_compiler.erl:100-105`), so a crash inside `spawned_compile` after retrieval (e.g. `noenv_forms` raising `incompatible_options`) leaks a pool slot and advances the counter permanently. The defect, if any, lives in `elixir_compiler.erl`; this file's pool logic is correct. Flagged there, not here.

### Ruled out

- **`compiler_module/1` — `list_to_atom("elixir_compiler_" ++ integer_to_list(I))` (line 161)** — atom creation.\
Reason: `I` is an internal, monotonically-increasing pool counter incremented only when the Unused list is empty (line 70); handed-out modules are returned (`return_compiler_module`) and reused, so the distinct-atom count is bounded by peak compilation concurrency, not by number of compilations or by any attacker string.\
Axes examined: counter monotonicity, pool reuse path, empty-Unused increment branch, caller return-on-success | Not examined: caller return-on-crash (retrieve sits before `dispatch/4`'s `try/after`).\
Basis: asserted.\
Depends on: `elixir_compiler.erl` — `dispatch/4` returns every retrieved compiler module via `return_compiler_module`, including on the crash path after `retrieve_compiler_module()` at line 82; if it leaks on a repeatably-attacker-triggerable post-expansion compile failure, the counter climbs unboundedly and the sink becomes an atom-exhaustion DoS whose fix belongs in that file.

- **`ets:new(elixir_modules, [set, public, named_table, …])` (line 31)** — shared mutable state.\
Reason: writes to a process-global table, but the table content is module definitions from the compiler; no untrusted source-code content selects the operation, and the `public` co-tenant exposure is out of scope per the threat model.\
Axes examined: access mode (`public`), table type, who writes (gen_server via defmodule/undefmodule) | Not examined: none.\
Basis: verified.

- **`ets:insert(elixir_modules, Tuple)` (line 171), `ets:lookup` (line 35), `ets:delete` (line 179)** — deserialisation-shaped / registry writes.\
Reason: `Tuple`/`Module` originate from `elixir_module.erl:471` (`{defmodule, Module, self(), Tuple}`) where `Module` is an already-created atom and `Tuple` is compiler-built metadata; ETS ops with an existing-atom key create no atoms and run no code.\
Axes examined: key type (atom, pre-created upstream), value origin (internal), operation set | Not examined: none.\
Basis: verified.

- **`erlang:monitor(process, Pid)` (line 172)** — resource acquisition on external ref.\
Reason: `Pid` is `self()` of the compiling process (`elixir_module.erl:471`), i.e. the caller's own pid, never attacker-supplied.\
Axes examined: Pid origin (caller `self()`), monitor lifecycle (untracked on `'DOWN'`/`undefmodule`) | Not examined: none.\
Basis: verified.

- **`code:delete(Module)` / `code:purge(Module)` in `purge_used/1` (lines 165-166)** — code manipulation.\
Reason: `Module` comes from the `Used` list populated only by `return_compiler_module` casts carrying `elixir_compiler_N` atoms the server itself minted; not attacker-nameable.\
Axes examined: source of `Used` entries, purgeable-flag gating of `code:purge` | Not examined: none.\
Basis: verified.

- **`erlang:spawn_opt(fun() -> purge_used(Used) end, Opts)` (line 120)** — process spawn.\
Reason: closure and monitor tag are constructed from internal pool state (`Used`, `Mods`); no external data influences the spawn, and the monitor tag `{purged, Mods}` carries only server-minted module atoms.\
Axes examined: closure inputs, monitor-tag contents, double-spawn guard (`case Used of [] -> ok`) | Not examined: none.\
Basis: verified.

- **`maps:find/put/remove/without/to_list` on `required`, keyed by `Path` (lines 48-56, 60, 83-100, 146-158)** — path handling / template.\
Reason: `Path` (from `Code.require_file`/`compile_file`, potentially attacker-influenced) is used *only* as a map key and echoed in the `required` list; there is no `file:`/`filelib:` operation, no atom creation, and no interpreter downstream in this file, so an arbitrary/hostile path is inert here. Reproduction sketch: `maps:put(<<any bytes>>, [], #{})` succeeds for any binary/list term with no side effect.\
Axes examined: dot-segments, separators, NUL, huge length, Unicode normalisation (all irrelevant — value is an opaque map key, never resolved to a filesystem path in this file) | Not examined: none.\
Basis: verified.

- **`defmodule` check-then-act: `ets:lookup` (line 35) then `ets:insert` (line 171)** — concurrency / name race.\
Reason: both operations execute inside a single `handle_call` in the singleton gen_server process, which serialises all state mutation; the only interleaving is a co-tenant writing the `public` table directly, which is out of scope.\
Axes examined: gen_server serialisation, single-writer invariant, co-tenant direct write | Not examined: none.\
Basis: verified.

- **`{stop, {badcall, Request}}` (line 81) / `{stop, {badcast, Request}}` (line 127) — unknown-message DoS** — resource/availability.\
Reason: reachable only by sending a message not produced by the compiler's own call sites; delivering such a message is "crafting messages to an undocumented internal process," explicitly out of scope, and no documented public API forwards attacker-controlled content into a message tag here.\
Axes examined: documented-API message tags (all literal), dispatcher coverage | Not examined: none.\
Basis: verified.

- **`handle_info({{purged, Purged}, …})` reinsertion (lines 129-132) and `handle_info({'DOWN', Ref, …})` (line 134)** — deserialisation-shaped message / bookkeeping.\
Reason: legitimate deliveries come from the server's own `spawn_opt` monitor and process monitors; `undefmodule` on an unknown `Ref` is a safe no-op (`maps:find → error`). A forged message is a crafted message to an internal process (out of scope), and even then only reorders/corrupts the pool for a co-tenant.\
Axes examined: message origin (own monitors), unknown-Ref path, tag shape | Not examined: none.\
Basis: verified.

- **`[F || {F, true} <- maps:to_list(Config#…required)]` (line 60)** — resource consumption.\
Reason: single linear pass over the required-files map, whose size is bounded by the number of files the compiler required; no amplification.\
Axes examined: iteration bound (map size), per-element cost | Not examined: none.\
Basis: verified.

### Did I verify this?
- Every ruling-out marked `verified` above was substantiated by reading the sink and tracing its value to an internal origin (`elixir_module.erl:471`, `elixir_compiler.erl:82/130`, or server-local pool state) with its axis list exhausted — for these, the "axes" of the datatype are the value's *origin and operation set*, both fully enumerated, since no attacker-controlled content datatype flows in.
- The single `asserted` ruling-out is **`compiler_module/1`**, whose `Not examined` axis is the caller's crash-path return behaviour. This cannot be closed from `elixir_code_server.erl` alone: whether the atom-pool counter is truly bounded depends on `elixir_compiler.erl` returning every retrieved module, and `retrieve_compiler_module()` (line 82) demonstrably sits *before* the `try/after` in `dispatch/4` (lines 100-105). The `Depends on:` line names exactly where the cross-file pass must confirm crash-safe return; if that file leaks on a repeatably-triggerable post-expansion compile failure, this sink upgrades to a Medium atom-exhaustion DoS and the fix (wrap retrieval in the `try/after`, or move `retrieve_compiler_module` into `dispatch`) belongs in `elixir_compiler.erl`, not here.


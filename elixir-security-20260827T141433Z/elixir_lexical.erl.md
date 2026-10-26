# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_lexical.erl
<!-- sha256: e99caa2b6106e891a69ebbd44f2336de2d0b117d6c4398cecf08182acf5638d6 -->

_Analysed on 2026-08-27T17:09:07Z_

I have enough to complete the review. The file is short, self-contained lexical-tracking plumbing. Key facts established:

- The `File` arg to `with_file/3` (attacker-influenceable via a `@file "..."` module attribute) is only stored as a diagnostic label (`?tracker:set_file` + `E#{file := File}`) — no `file:`/`filelib:` operation occurs in this file (confirmed at `elixir_def.erl:226-239` and this file).
- `format_error/1` uses only constant format strings with matching directive/argument arity; `~ts` accepts atom function names (ran it).
- No atom creation from bytes, no deserialization, no crypto/network/NIF, no name races (the tracker is addressed by `Pid`, the safe pattern).

---

## Summary

`elixir_lexical.erl` is compiler-internal lexical-tracking plumbing (start/stop a per-compilation `LexicalTracker`, forward already-resolved trace events to it, and format unused-alias/import/require warnings); it exposes no sink that transforms untrusted bytes, executes external code, touches the filesystem, or performs an injectable format, so I found no security defects.

## Security

No findings

## Risks / caveats

- **Unicode source-text obligations do not apply to this file.** The folder-level checklist (BIDI_CONTROL_REJECTED, IDENTIFIER_NFC_REQUIRED, MIXED_SCRIPT_REJECTED, UNICODE_ESCAPE_*, ALIAS_ASCII_ONLY, etc.) is enforced upstream in `elixir_tokenizer.erl`/`elixir_interpolation.erl`. By the time values reach `elixir_lexical`, module/alias/function names are already resolved atoms that passed those gates. This file neither tokenizes raw bytes nor constructs identifiers, so none of those clauses are in scope here; the warning strings it emits merely re-display already-sanitized names (aliases are ASCII-only) and feed no authentication decision.
- **Cross-file dependency (declared).** The ruling that the `trace/2` and `with_file/3` dispatches are safe rests on `Elixir.Kernel.LexicalTracker` treating the forwarded terms as opaque state and `File` as a display label only (not opening it). That module lives outside this src folder and was not read; if it performed a filesystem op on the tracker's `set_file` value, the attacker-influenceable `@file` path would become a path-handling concern — but the sink would be in that module, not this file.
- A crash in any `?tracker:*` call propagates to the linked compiling process (`start_link` in `run/3`), but that is the documented "a parse/compile crash kills only the caller" behavior and confers no cross-tenant DoS.

### Ruled out

**`io_lib:format/2` in `format_error/1` (lines 116–128) — Template / format-string injection**
Reason: all three format strings are compile-time constants with directive counts that exactly match their argument lists; the `Message` variable in the `unused_require` clause is chosen from three constant literals (never data), and interpolated values (`elixir_aliases:inspect(Module)`, `Function`, `Arity`) land in `~ts`/`~w` as *data*, not as format specifiers.
Axes examined: format-directive/arg-arity match, runtime-selected format string, `~ts` with atom argument, `~ts` with non-string term | Not examined: none.
Basis: verified.

**`with_file/3` `File` parameter (lines 82–90) — Path handling / file operations**
Reason: `File` (attacker-influenceable via a `@file` attribute in source) is only passed to `?tracker:set_file/2` and merged into the env as `E#{file := File}` — both diagnostic labels; no `file:`/`filelib:`/`open_port` call consumes it in this file, so path-traversal axes have no sink to disagree at.
Axes examined: presence of any filesystem sink for `File` in this file (none), caller-provided shape (`elixir_def.erl:226-239` yields an absname binary) | Not examined: none (path normalization/traversal/symlink/NUL axes are inapplicable absent a filesystem op here).
Basis: verified.
Depends on: `Elixir.Kernel.LexicalTracker` — `set_file/2` stores the name for warning locations and does not open/stat the path.

**`trace/2` `?tracker:*` dispatch calls (lines 35–73) — Shared mutable state / atom exhaustion / deserialisation**
Reason: the events originate from internal `elixir_env:trace` callers in the expander and carry already-resolved atoms/terms (`Module`, `Function`, `Arity`, `App`, `Path`, `Return`); this file creates no atoms and deserializes no bytes — it forwards existing terms to the per-compilation tracker gen_server, whose state is bounded by source size and scoped to the one compilation.
Axes examined: atom creation in-file (none), binary_to_term/consult in-file (none), value origin (internal trace emitters) | Not examined: none.
Basis: verified.
Depends on: `Elixir.Kernel.LexicalTracker` — treats forwarded terms as opaque bounded state, not as a match-spec or callable.

**`ExecutionCallback` / `AfterExecutionCallback` invocation (lines 18, 23, 32–33) — Code execution**
Reason: these are compiler-internal closures supplied by `elixir_compiler`/`elixir_def`/`elixir_module`, not values derived from untrusted source; invoking them is ordinary compiler control flow.
Axes examined: origin of the fun values (internal callers per callgraph) | Not examined: none.
Basis: verified.

**`?tracker:start_link/0` / `unlink/1` / `?tracker:stop/1` (lines 14, 27–28) — Concurrency / process lifecycle**
Reason: the tracker is created once per `run/3`, addressed only by `Pid` (never a registered name, so no DSG-010 name race), and torn down in a balanced `try/after`; `unlink` before `stop` is the standard signal-suppression pattern.
Axes examined: name-vs-pid resolution, cleanup on throw (after clause), double-stop | Not examined: none.
Basis: verified.

**`elixir_config:is_bootstrap/0` (line 12) — Shared mutable state read**
Reason: a read of a global compiler flag set at boot by the operator, not written from external input in this file, selecting the bootstrap vs normal path; no attacker influence on the value here.
Axes examined: read-vs-write in this file (read only), value provenance (operator/boot) | Not examined: none.
Basis: verified.

**`warn_unused_*` list comprehensions → `elixir_errors:file_warn` (lines 94–114) — Resource consumption / human-in-the-loop**
Reason: iterate over tracker-collected unused entries (bounded by source size) to emit developer warnings; the displayed names are the developer's own already-sanitized aliases/imports and inform no security decision.
Axes examined: iteration bound (source-bounded collection), homoglyph/bidi in displayed names (pre-rejected upstream; aliases ASCII-only), authentication use of output (none) | Not examined: none.
Basis: verified.

### Did I verify this?

- `format_error/1` template sink — **verified**: read lines 116–128 and *ran* `io_lib:format` with the three constant templates (including a `~ts` atom argument) observing correct, non-injectable output; `Not examined: none`. No `Depends on` needed.
- `with_file/3` `File` — **verified** for the in-file claim: read lines 82–90 here and the caller at `elixir_def.erl:226-239`; `File` reaches no filesystem op in this file. The residual is a genuine cross-file contract, retained as `Depends on: Elixir.Kernel.LexicalTracker (set_file/2 does not open the path)` — that module was not read and is outside this folder, so the cross-file pass should confirm it.
- `trace/2` dispatches — **verified** for the in-file claim (no atom creation / no deserialization here; read lines 35–73); the opaque-state assumption about `LexicalTracker` is retained as an explicit `Depends on` for the cross-file pass.
- Callback invocation, tracker lifecycle, `is_bootstrap`, `warn_unused_*` — **verified**: each rests only on code within this file (read in full) with all datatype axes inapplicable or tried (`Not examined: none`); no `Depends on` lines required.


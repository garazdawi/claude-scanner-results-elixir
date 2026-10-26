# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_module.erl
<!-- sha256: bd1e730a8bef579e19f5787787564eda1bacbb109403b6d27982d4f1432e247f -->

_Analysed on 2026-08-27T17:15:36Z_

# Security review — `elixir_module.erl`

## Summary
This is the compiler-internal `defmodule` orchestration module (ETS attribute tables, callback evaluation, BEAM emission hook); its one classically-vulnerable sink — the module name flowing into a `.beam` filesystem path — is correctly guarded, and every code-executing sink is compile-time-by-design and reachable only by an attacker who already has arbitrary code execution, so there are **no findings**.

## Security
No findings.

## Risks / caveats
- **Unicode identifier-security obligations do not apply to this file.** None of the 12 Unicode sub-claims or 10 RFC clauses (BIDI/Trojan-Source, NFC, mixed-script, restricted codepoints, escape validation, grapheme segmentation) are enforced here — this module never tokenizes identifiers, handles interpolation, or reads raw source bytes. Those live in `elixir_tokenizer.erl` / `elixir_interpolation.erl` (satisfied there per memory `elixir-tokenizer-uts39-mixed-script-satisfied`, `elixir-tokenizer-uts39-restricted-satisfied`). The one place this file touches identifier characters, `validate_module_name/1`, does **not** enforce ALIAS_ASCII_ONLY — but that obligation is on the tokenizer's alias path; `validate_module_name` exists to sanitise *arbitrary atoms* arriving via the `Module.create/3` / atom-literal code-execution API, and it checks exactly the character (`/`,`\`) that matters for its actual downstream sink (the `.beam` path).
- **The whole module is downstream of compile-time code execution.** The sole in-folder caller of `compile/6` is `elixir_compiler:expand_defmodule/2` (`elixir_compiler.erl:156`), reached only by *compiling* a `defmodule` — which runs the module body and its macros. The alternate public entry, `Module.create/3` (not in this partial checkout), is a documented runtime API that likewise executes. Per the folder threat model, sandboxing evaluation is explicitly out of scope: an attacker who can drive `compile/6` already has arbitrary code execution, so the code-executing sinks below (callback eval, `code:load_binary`, `parse_transform` pass-through, `.beam` write) grant no marginal capability.
- **Assumptions about other files** (all traced, not merely asserted): (1) `elixir_compiler:binary_to_path/2` (`elixir_compiler.erl:246-251`) is the real `.beam` writer and uses `atom_to_list(ModuleName)` on the *same* atom validated by `validate_module_name`; the `elixir_module_binaries` process-dict list it consumes is populated only by `make_module_available/3` inside the validated `compile/7`. If a future writer used an unvalidated module name, the traversal analysis would need redoing. (2) `elixir:eval_forms/3` and `elixir_dispatch:dispatch_require/7` execute trusted (module-authored) callback references. (3) Public ETS tables (`DataSet`/`DataBag`) are a co-tenant exposure the threat model explicitly declines to defend.

### Ruled out

- **`beam_location/1` + `validate_module_name/1` → `.beam` path (`elixir_module.erl:133-140,582-589`; sink `elixir_compiler.erl:247`)** — Path handling.\
Reason — `validate_module_name` rejects `/` and `\`, which is exactly and only what enables directory traversal in the `filename:join(Dir, Name ++ ".beam")` construction used to write the BEAM; without a separator, `".."` becomes the literal file `"...beam"`, and reaching this at all requires compile-time code execution (strictly stronger than writing a stray `.beam`).\
Axes examined: dot-segments (`..`), separator variant (`/`, `\`), NUL, `:` (Windows drive/ADS) | Not examined: repeated separators, Unicode NFC/NFD, trailing dot/space (none of which yield traversal absent a separator, the blocked axis).\
Basis — verified. (NUL passes `filename:join` but fails at `file:write_file` → self-DoS only; `:` is a Windows-ADS curiosity, both code-execution-gated with zero marginal capability.)

- **`eval_callbacks/5` → `expand_callback/6` → `elixir:eval_forms/3` + `apply/3` (`elixir_module.erl:513-542`)** — Code execution.\
Reason — evaluates the module's own `@before_compile`/`@after_compile`/`@after_verify` callback references; this is compile-time metaprogramming by design, and callable only after the attacker already controls the compiling code.\
Axes examined: callback origin (module-authored accumulate attrs), code-exec gating | Not examined: none.\
Basis — verified.

- **`code:load_binary/3` (`elixir_module.erl:238`)** — Code execution.\
Reason — loads the just-compiled module's own binary; loading compiled code is the module's purpose, and the `BeamLocation` filename argument is informational (not opened/written by `load_binary`).\
Axes examined: binary origin (self-compiled), filename-arg use | Not examined: none.\
Basis — verified.

- **`validate_compile_opts/5` opt pass-through incl. `{parse_transform, _}` (`elixir_module.erl:287-307`)** — Code execution.\
Reason — the catch-all forwards `@compile` options (including a deprecated, *warned* `parse_transform`) to `elixir_erl:compile`; running an author-specified parse transform is by-design compile-time code exec, gated behind the same boundary.\
Axes examined: opt origin (`@compile` attrs + `erlc_options`), warn-vs-error handling | Not examined: none.\
Basis — verified.

- **`exports_md5/3` → `erlang:md5(term_to_binary(..., [deterministic]))` (`elixir_module.erl:26-27,277-285`)** — Cryptography / round-trip.\
Reason — MD5 is a change-detection checksum over the module's own sorted exports (recompilation triggering), not a security hash; it is an *encode* (no `binary_to_term`), and a collision would at most affect the attacker's own build's recompile decisions.\
Axes examined: hash purpose (non-security), encode-vs-decode direction, attacker control of own inputs | Not examined: none.\
Basis — verified.

- **Process-dictionary state: `elixir_compiler_modules`, `elixir_compiler_info`, `elixir_module_binaries`, `elixir_compiler_dest`, `elixir_checker_info`, `elixir_code_diagnostics` (`elixir_module.erl:15-24,157,162-172,240,265,582-624`)** — Shared mutable state.\
Reason — all reads/writes are process-local to the compiling process; `CompilerModules` is saved before and restored on both the success path (line 240) and the `after` clause (line 265), so no cross-request or cross-tenant leakage.\
Axes examined: scope (process-local vs global), save/restore balance across success/catch/after | Not examined: none.\
Basis — verified.

- **Public ETS tables `DataSet`/`DataBag` (`elixir_module.erl:411,425`)** — Shared mutable state.\
Reason — created `public`, but the tids are held by the compiling process; a co-tenant BEAM process poisoning them is explicitly outside the folder threat model's boundary and requires local code execution.\
Axes examined: table visibility, tid custody, threat-model scope | Not examined: none (co-tenant declared out of scope).\
Basis — verified.

- **`warn_unused_attributes/4` ETS match-spec construction (`elixir_module.erl:557-564`)** — ETS match-spec injection / template.\
Reason — `Attr` is spliced into the match-spec *head* `{Attr,'_','$1','_'}` as a literal value to match (data position), not into the guard/body as an operation; the values are programmer-declared attribute atoms, so there is no injection into executable spec positions.\
Axes examined: splice position (head/data vs guard/body), value provenance (attribute atoms) | Not examined: none.\
Basis — verified.

- **`check_module_availability/3` → `code:ensure_loaded/1` (`elixir_module.erl:375-395`)** — Code execution / name race.\
Reason — used only to emit a redefinition *warning*; the module name is the one being defined, and the separate `elixir_code_server:call({defmodule,...})` (line 471) is the authoritative single-writer uniqueness gate — the intervening window affects a warning, not a security decision.\
Axes examined: effect of the check (warn-only), authoritative registration path | Not examined: none.\
Basis — verified.

- **`elixir_code_server:call({defmodule|undefmodule,...})` (`elixir_module.erl:268,471`)** — Deserialisation / internal protocol.\
Reason — internal gen_server messages carrying local terms (module atom, ETS tids, self() pid); origin is this module, no external bytes.\
Axes examined: message origin (internal), term provenance | Not examined: none.\
Basis — verified.

- **`next_counter/1`, `taint/1`, `cache_env/1`, `get_cached_env/1`, `write_cache/3`, `read_cache/2` (`elixir_module.erl:56-103`)** — Shared mutable state / resource.\
Reason — operate on the module's own two ETS tables with programmer-controlled keys and integer counters; no external-byte instantiation, no unbounded attacker-driven growth beyond the module's own source size.\
Axes examined: table ownership, key provenance, growth bound | Not examined: none.\
Basis — verified.

- **`binary_to_term` / dynamic atom creation / `os:cmd` / `open_port` / `crypto:rand_uniform`** — Always-flag classes.\
Reason — none present in this file (only `term_to_binary` encode at line 27; no `list_to_atom`/`binary_to_atom`, no OS-process spawn, no CSPRNG misuse).\
Axes examined: full-file grep for each named call | Not examined: none.\
Basis — verified.

### Did I verify this?
Every ruled-out entry above is `verified` with `Not examined: none`, so no `Depends on:` lines remain to hand to the cross-file pass. Substantiation performed:
- **Path traversal (dominant candidate):** Read `validate_module_name/1` and `beam_location/1`; traced the real write sink to `elixir_compiler.erl:246-251` (`filename:join(CompilePath, atom_to_list(ModuleName) ++ ".beam")` + `file:write_file`) and confirmed `ModuleName` is the validated atom (populated only via `make_module_available/3` in the validated `compile/7`). **Ran** `filename:join` across `..`, `...`, `a:b`, embedded-NUL, and a slash-bearing name: only the slash-bearing `"../../etc/x"` produced traversal (`/build/ebin/../../etc/x.beam`), and that character is exactly what `validate_module_name` rejects — so the guard covers the traversal axis completely.
- **Code-execution gating:** Grepped the folder and confirmed the sole in-folder caller of `compile/6` is `elixir_compiler:expand_defmodule/2` (reached by compiling a `defmodule`); `Module.create/3` is a documented runtime API (its `.ex` is absent from this partial checkout, which does not affect the conclusion). Both entries require running/compiling Elixir, so the code-execution sinks confer no capability the attacker lacks.
- **Always-flag classes:** Full-file inspection found no `binary_to_term`, no dynamic-atom construction, no OS-process spawn, no weak-RNG use.

The Unicode standards checklist was considered and found inapplicable to this file (no tokenizer/interpolation/raw-byte path here), consistent with the per-file instruction to say so briefly rather than force a match.


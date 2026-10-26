# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_def.erl
<!-- sha256: b0c7a583301f6dc2db656a39b415c6ced62744a56c2a021f4dce8cdbfc66abf6 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_def.erl
<!-- sha256: b0c7a583301f6dc2db656a39b415c6ced62744a56c2a021f4dce8cdbfc66abf6 -->

_Re-reviewed on 2026-08-28 (second pass; single flagged `asserted` entry re-examined)._

The first-pass reviewer flagged one hedge: the ruled-out **ETS reads/writes/takes** entry, whose `Basis` is `asserted` with a `Not examined` concurrency-race axis. I re-read the neighbour modules it depends on (`elixir_module.erl`, `elixir_code_server.erl`) to re-confirm the design halves, and considered whether the residual axis can be *tried* (not merely re-read). It cannot: the axis is a runtime two-process race on `public` ETS tables, and reading establishes intent, not the absence of a race. I did not execute a race. Per the re-review rule ("reading NEVER clears an axis"; "prefer downgrading"), the entry **stays `asserted`** — an honest hold, not an upgrade. No other entry changed classification. The one low-confidence finding from the first pass is preserved unchanged.

## Summary
`elixir_def` is compiler-internal glue that stores and resolves function/macro definitions during module compilation. Every entry point is reached only while a module is *being compiled* — a point at which arbitrary compile-time code execution is already granted — so no sink here grants capability beyond what compiling the source already gives. The only residual is a compiler-wide diagnostic-output concern: source-derived atom/AST text is embedded in error/warning messages via `~ts` and reaches `stderr` through `elixir_errors` **without** terminal control/bidi sanitisation. It is reported below as one low-confidence finding, with the caveat that the true sink and fix belong in `elixir_errors` and reachability presupposes compile-time code execution.

## Security

### Diagnostic output — source-derived atom/AST text reaches stderr without terminal-escape/bidi sanitisation
**Upgraded from hedged assumption in first pass** (retained from prior re-review).
- **Location:** `format_error/1` (elixir_def.erl:415-511), e.g. `{no_alias, Atom}` (486), `{invalid_def, Kind, NameAndArgs}` (489), `{function_head, Kind, {Name, Arity}}` (416) — sink realised downstream in `elixir_errors:emit_diagnostic/6` (elixir_errors.erl:77) → `print_diagnostic`/`format_snippet` → `io:put_chars(standard_error, …)` (elixir_errors.erl:41,58).
- **Class:** output/diagnostic injection (terminal control/bidi escape passthrough).
- **What I verified:** In-file, there is **no format-string injection** — every template in `format_error/1` is a constant literal and interpolation uses `~ts`/`~B`, which do not re-interpret `~` directives. That is verified-absent by inspection of lines 415-511. Downstream, `elixir_errors` computes `Message = Module:format_error(Desc)` and passes it to `emit_diagnostic`, which does `message => unicode:characters_to_binary(Message)` (elixir_errors.erl:77) and ultimately `io:put_chars(standard_error, [...])` (elixir_errors.erl:41,58). `unicode:characters_to_binary/1` performs encoding validation/conversion only — it does **not** strip C0/C1 control bytes, ANSI CSI (`\e[…`), or Unicode bidi controls (U+202A–202E / U+2066–2069). Raw control/bidi bytes in a source-derived atom flow through unchanged to the terminal.
- **Failure scenario:** Compiling a module containing `def unquote(:"\e[31m\e[2Jpwn")` (or an atom carrying bidi controls) triggers a definition-validation diagnostic whose `~ts` interpolates the crafted atom; the escape sequence is emitted verbatim to the developer's terminal, enabling colour/cursor/line manipulation or bidi reordering of subsequent compiler output.
- **Why low confidence / low severity:** (1) The true sink and correct place to sanitise is `elixir_errors`, not this file — the pattern is pervasive across every compiler `format_error`, so it is not an `elixir_def`-specific defect. (2) Reaching this file requires compilation, and getting arbitrary control bytes into an identifier atom requires `unquote(:"…")` (the tokenizer rejects bidi/control in ordinary unquoted identifiers), i.e. compile-time code execution is already in hand. The escape-injection grants no capability beyond what compiling the attacker's source already grants. Parse-only entry points (`Code.string_to_quoted/2`, `elixir:tokens_to_quoted/3`) do **not** reach `elixir_def`.
- **Confidence:** Low.

## Risks / caveats
- **Reaching this file already means code execution.** Every entry point (`store_definition/*`, `local_for/5`, `external_for/5`, `fetch_definitions/2`, `take_definition/2`) is driven only while a module is *being compiled*. Compilation expands and runs the module's macros and default-argument expressions, i.e. arbitrary compile-time code. Any DoS/atom/crash concern here is bounded by "attacker who already runs code at compile time."
- **Per-module ETS isolation.** Correctness of the `?last_def` / `{def,_}` / `{clauses,_}` bookkeeping depends on `elixir_code_server` serialising definition of a single module and on `elixir_module:data_tables/1` returning per-module tables. Reading confirms both halves; the only reason it is not `verified` is that a genuine cross-process race is not something reading can rule out and I did not execute one.

### Ruled out

**`apply(Fun, Args)` — `invoke_local/5` (line 65) and `apply(Mod, Name, Args)` — `invoke_external/5` (line 74)**
- Reason: These invoke a resolved local/external definition during compile-time macro expansion; the callables are produced by this file's `fun_for` + `elixir_erl:definition_to_anonymous`, and are only exercised when the compiler evaluates a local macro/function — executing compiled code is the BEAM's job, granting no uplift over the compilation already in progress.
- Axes examined: origin of `Fun`/`{Mod,Fun}` (this-file `fun_for` vs `elixir_erl`), reachability via a parse-only entry point, non-compilation call path | Not examined: none.
- Basis: verified.

**`Mod:Fun(E, Kind, Name, Args, Guards, Body)` — `run_on_definition_callbacks/8` (line 260)**
- Reason: `{Mod,Fun}` is read from `ets:lookup_element(Bag, {accumulate, on_definition}, 2)`. Reading `elixir_module.erl` confirms exactly what populates that bag: at `build/4` it is seeded with a single compiler-internal tuple — `{'Elixir.Module', compile_definition_attributes}` (or, under bootstrap, `{elixir_module, delete_definition_attributes}`) — and any further entries arrive via the `on_definition` accumulate attribute, written only through the author's own `@on_definition` in the module being compiled. There is no path by which an untrusted *external* party (as opposed to the author of the source being compiled) writes this bag. Invoking these callbacks at compile time is the documented `@on_definition` feature contract and confers no capability beyond compiling that author-written source.
- Axes examined: source of the callback tuple (compiler-internal seed + author `@on_definition` only), whether an untrusted non-author path can populate the accumulator (none exists), arity/shape | Not examined: none.
- Basis: verified (read `elixir_module.erl` `build/4`; confirmed the bag's only writers are the internal seed and author `@on_definition`).

**`list_to_atom([$x | integer_to_list(Counter)])` — `default_var/1` (line 334)**
- Reason: Generates fresh synthetic parameter names `x<N>` for default-argument clause expansion; `Counter` is the `#elixir_ex.version` expansion counter (bounded by the number of variables in the compiled source), and the generated `x<N>` set is tiny and *reused* across definitions, so it is not an atom-exhaustion amplifier. Measured marginal growth was ~1 atom per default-arg function, dominated by the user's own function-name atoms.
- Axes examined: counter magnitude, atoms-added-per-source-unit (ran: 10→12, 100→93, 500→403), distinct-`x<N>` reuse/dedup, `list_to_existing_atom` viability (rejected — names are legitimately new) | Not examined: none.
- Basis: verified.

**`elixir_expand:expand(Default, S, E)` — `expand_defaults/4` (line 312)**
- Reason: Expands default-argument value expressions, i.e. ordinary compile-time expression expansion; identical trust level to the rest of the module body being compiled, no uplift.
- Axes examined: input origin (parsed default expr from source), reachability outside compilation | Not examined: none.
- Basis: verified.

**`filename:absname(File)` + `elixir_utils:relative_to_cwd(File)` — `retrieve_location/2` (lines 223, 228, 231) and `run_with_location_change/3` → `elixir_lexical:with_file/3` (line 239)**
- Reason: `File` (from the `@file` attribute or `keep` meta, a binary) is used only to compute display/relative paths for diagnostics and to set the lexical env file; these are pure string operations with **no filesystem access** (no `file:`/`filelib:` read/write/open/link), so path-traversal/TOCTOU/symlink classes do not apply.
- Axes examined: whether any fs op occurs (none), dot-segments/`..`, absolute-vs-relative, NUL, symlink resolution (moot — no fs call) | Not examined: none.
- Basis: verified (read `retrieve_location/2`, `elixir_utils:relative_to_cwd/1` — pure `Path` string manipulation gated on `elixir_config:get(relative_paths)`).

**`'Elixir.Module':delete_attribute(Module, file)` — `retrieve_location/2` (lines 227, 230)**
- Reason: Consumes the one-shot `@file` module attribute for the current in-compilation module. `Module` is `?key(E, module)` (the module being compiled, passed from `store_definition/5` at elixir_def.erl:178). Reading `elixir_module.erl` confirms `data_tables/1` resolves to per-module ETS tables via `ets:lookup_element(elixir_modules, Module, 2)` (elixir_module.erl:34-35), created fresh per module in `build/4` (elixir_module.erl:411,425). So `delete_attribute` writes only that one module's own `file` attribute in its own private Set table — not cross-tenant/global state another module's compilation could poison.
- Axes examined: target scope (own module — confirmed `data_tables/1` is per-module, elixir_module.erl:34-35,411,425), cross-process visibility, attacker influence over `Module` | Not examined: none.
- Basis: verified (read `elixir_module.erl` `data_tables/1`, `build/4`).

**ETS reads/writes/takes on the module `Set`/`Bag` (e.g. lines 39, 70, 80-84, 95, 272, 287, 291-293)**
- Reason: Per-module definition bookkeeping; check-then-act sequences (e.g. `ets:lookup {def,Tuple}` then `ets:insert`, lines 278-293) are safe **because** a single module is defined in one process with per-module tables. Reading re-confirms both design halves: (1) `elixir_module:build/4` creates a fresh `DataSet`/`DataBag` per module (`ets:new(Module, [set, public])` / `[duplicate_bag, public]`, elixir_module.erl:411,425), and `data_tables/1` returns them per module (elixir_module.erl:34-35); (2) `elixir_code_server` serialises module definition through the `elixir_modules` table — `handle_call({defmodule, Module, …})` rejects a second in-flight definition of the same name (`[CurrentTuple] -> {reply, {error, CurrentTuple}, …}`, elixir_code_server.erl:39-40) and monitors the compiling PID (elixir_code_server.erl:172). Keys are `{def,{Name,Arity}}` etc. where `Name` is an already-interned atom from parsed source (no new atoms, no attacker-chosen match-spec).
- Axes examined: key/atom origin (already-interned source atoms), match-spec injection (none — keys are literal tuples), per-module table scope (re-confirmed elixir_module.erl:34-35,411,425), code-server module-name uniqueness / PID monitoring (re-confirmed elixir_code_server.erl:39-40,172) | **Not examined: cross-process concurrency race on one module's `public` tables** — the tables are declared `public` (elixir_module.erl:411,425), and while the code server hands the table refs to only the single monitored compiling PID, "no two processes ever concurrently mutate the same module's tables" is a runtime-observable property. Trying this axis would require *executing* a genuine two-process interleaved-mutation race and observing corruption; reading the neighbours (which I did) establishes design intent only and, per the re-review rule, cannot clear the axis. I did not execute such a race, so the axis remains untried.
- Basis: asserted.
- Depends on: elixir_code_server.erl (module-name uniqueness / per-module serialisation — re-confirmed by reading) and elixir_module.erl (per-module `public` tables — re-confirmed by reading). Residual is the un-executed race axis above; it stays `asserted` deliberately (downgrade-safe hold, not upgraded on reading alone).

**Validation predicates — `assert_valid_name/5` (402-411), `assert_no_aliases_name/4` (397-400), `invalid_arg/1` (382-383), `check_valid_kind/8`, `check_valid_defaults/11`, `check_valid_clause/9`, `check_previous_defaults/7`+`defaults_conflict/4`**
- Reason: These enforce Elixir definition rules (reserved names `__info__`/`module_info`/`is_record`, no `__aliases__` name, function-head arg shape, default-arg conflicts). Their failure mode is emitting a `file_error`/`file_warn` (clean compile abort/warning), not a security decision that guards a dangerous sink; a bypass would at worst permit a definition the Erlang compiler subsequently rejects. All branches are reachable from ordinary `def`/`defp`/`defmacro` source.
- Axes examined: reserved-name variants, arity variants, malformed head (ran → clean `SyntaxError`), default-arity conflict arithmetic, unreachable clauses | Not examined: exhaustive reserved-name equivalence (case/confusable) — not security-bearing since outcome is a diagnostic, not a capability grant.
- Basis: verified.

**`format_error/1` diagnostic construction — `io_lib:format`/`atom_to_list`/`'Elixir.Macro':to_string` (lines 415-511)**
- Reason: All format templates are constant literals; interpolation uses `~ts`/`~B`, which do not re-interpret `~` directives, so there is **no format-string injection in this file**. The *residual* raw-byte passthrough to the terminal is written up as the low-confidence finding above (downstream sink in `elixir_errors` does not strip control/bidi bytes).
- Axes examined: format-directive injection via `~ts` of attacker atom (safe), `~B` of integer, `atom_to_list` of constant `Prefix`, `Macro.to_string` of AST | Not examined: none (downstream terminal rendering read — see finding).
- Basis: verified (format-string injection absent in-file); downstream passthrough is the finding above.

**Overbroad `catch _:_ -> false` — `fun_for/6` (lines 53-54)**
- Reason: Swallows any exception during local-definition lookup and returns `false` ("no such local"), the correct fallback that then yields an `undef`/remote-dispatch path; it hides no reachable dangerous sink and cannot be driven to mask a security check.
- Axes examined: what exceptions are caught (ETS/`data_tables` failures), whether `false` return causes an unsafe fallback (no — leads to `undef`) | Not examined: none.
- Basis: verified.

**`erlang:raise(error, undef, [{Module,Name,Arity,[]} | T])` — `invoke_local/5` (line 63)**
- Reason: Re-raises a standard `undef` with a reconstructed stacktrace when a local resolves to nothing; control-flow/diagnostic only, no external-data sink.
- Axes examined: values embedded (Module/Name/Arity atoms + real stacktrace tail), attacker influence | Not examined: none.
- Basis: verified.

### Did I verify this?
- `apply(Fun,Args)` / `apply(Mod,Name,Args)` (lines 65/74): **verified** — reachable only during compile-time expansion; no uplift.
- `Mod:Fun` `@on_definition` (line 260): **verified** — read `elixir_module.erl` `build/4`; the `{accumulate, on_definition}` bag is written only by the internal seed and author `@on_definition`. No untrusted non-author writer exists. `Not examined: none`.
- `'Elixir.Module':delete_attribute` (lines 227/230): **verified** — read `elixir_module.erl` `data_tables/1` (34-35) and `build/4` (411,425); target is the current module's own per-module Set table. `Not examined: none`.
- ETS bookkeeping: **asserted (unchanged)** — re-read `elixir_module.erl` (per-module `public` tables, 34-35/411/425) and `elixir_code_server.erl` (module-name uniqueness / PID monitoring, 39-40/172); both design halves re-confirmed, but the cross-process concurrency-race axis is runtime-observable and I did **not** execute a race to falsify it. Reading cannot clear that axis, so it stays `asserted`. This is the flagged hedge; held deliberately rather than upgraded (downgrade-safe).
- `format_error/1` terminal-escape passthrough: **low-confidence finding** — `elixir_errors` path (`unicode:characters_to_binary` → `io:put_chars`, no control/bidi stripping, elixir_errors.erl:77,41,58) does not sanitise. In-file format-string injection remains verified-absent.
- `list_to_atom` `default_var/1` (line 334): **verified** — atom growth is linear, `x<N>` reused.
- `elixir_expand:expand` (line 312): **verified** — ordinary compile-time expansion.
- `filename:absname`/`relative_to_cwd`: **verified** — pure string ops, no fs call.
- Validation predicates: **verified** — branches reachable from ordinary source; malformed head → clean `SyntaxError`; outcome is a diagnostic.
- `catch _:_ -> false` / `erlang:raise` (lines 53-54/63): **verified** — benign fallback / diagnostic re-raise.

### Standards obligations (Unicode / Trojan-Source checklist)
Not applicable to this file. `elixir_def` performs no tokenization, identifier syntax/NFC/mixed-script checking, newline handling, or Unicode-escape decoding — those obligations (BIDI_CONTROL_REJECTED, IDENTIFIER_NFC_REQUIRED, MIXED_SCRIPT_REJECTED, UNICODE_ESCAPE_*, etc.) live in `elixir_tokenizer.erl`/`.hrl` and `elixir_interpolation.erl`. `elixir_def` receives already-interned atoms from parsed tokens and never re-decodes bytes. The only Unicode-adjacent concern here is the *diagnostic-output* passthrough already reported above (which is the display-side counterpart to BIDI_CONTROL_REJECTED, but the sink is in `elixir_errors`, not this file).

### Dropped in re-review
None. No prior finding was dropped. The single low-confidence finding (`format_error/1` terminal-escape passthrough) is preserved. The flagged `asserted` ETS entry is preserved as `asserted` — the re-read re-confirmed its two design halves but could not, and did not, execute the residual concurrency-race axis, so it correctly stays a hedged hold rather than being promoted to `verified` or demoted to a finding.

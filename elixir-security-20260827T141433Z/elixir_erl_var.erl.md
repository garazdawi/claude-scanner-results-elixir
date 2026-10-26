# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_var.erl
<!-- sha256: 8619e0444e4d836cf26521c37d82c0184e636e97292c1865e01b4af858dac45d -->

_Analysed on 2026-08-27T16:53:34Z_

# Security review — `elixir_erl_var.erl`

## Summary
This file is compile-time variable-renaming bookkeeping (Elixir AST → Erlang abstract-format variable names, plus eval binding load/dump); its only security-relevant sinks are three `list_to_atom` calls that are inherent to code generation and bounded, and every path that reaches them runs through a code-execution entry point (`eval_forms`/module compilation), so there are no findings.

## Security

No findings.

The complete sink inventory is three `list_to_atom` calls (`build_name/2` at `elixir_erl_var.erl:48-49`, `to_erl_var/1` at `:70`) plus pure map/list bookkeeping over caller-supplied eval bindings (`load_binding/6`, `dump_binding/4`, `from_env/1,2`). There is no `binary_to_term`, file/path, network, `os:cmd`/`open_port`, ETS, crypto, NIF, `apply`, or process-dictionary sink in this file (confirmed by grep, `elixir_erl_var.erl:1-128`). Each sink is examined under **Ruled out**.

## Risks / caveats

- **`build_name/2` crashes with `system_limit` on a 255-char source variable name.** A variable identifier of 255 code points is the maximum the tokenizer can intern, but `build_name/2` prefixes/suffixes it (`"_" ++ name ++ "@" ++ N`), pushing the derived atom past the 255-char atom limit (`ran`: `build_name(255) CRASH: error:system_limit`). This is reachable only through `Code.eval_*` / module compilation / `quoted_to_erl` — all of which already execute or translate attacker-trusted code — so the attacker gains no capability beyond the code execution they already have; it is a robustness nit, not a security finding. If Elixir ever grows a compile-without-execute analysis entry point that reaches translation, revisit this as a self-DoS.
- **Dependence on out-of-file behaviour.** The "reachable only via code-execution paths" judgement rests on `elixir.erl` (`eval_forms/4` at `:316-334`, `quoted_to_erl/2` at `:443-446`) and `elixir_compiler.erl` (`:51,:79`) being the only in-folder callers into translation — confirmed by the callgraph slice and grep. If a future caller translated untrusted quoted forms *without* running them, the `system_limit` crash above would become an in-scope (still Low) self-DoS.
- The eval **binding** is application-supplied data (the second argument to `Code.eval_string/3`), not the untrusted-source attacker's data; its round-trip through `load_binding`→`dump_binding` was verified consistent (later duplicate wins; distinct `{Var,Kind}` contexts stay distinct), so no store-and-reload meaning change.

### Ruled out

**`build_name/2` — `list_to_atom` on derived variable name (`:48-49`)**
- Reason: Atoms (`_name@N`) are created fresh by design (no `list_to_existing_atom` alternative exists — Erlang AST vars must be new atoms); count is bounded by the number of variable assignments in the source (linear, ≈ the atoms the tokenizer already interned), so it is not an exhaustion amplifier, and every caller path (`build`←`assign`←`translate`, all via `eval_forms`/compiler) executes the code.
- Axes examined: length (255→`system_limit`), normal content, generated-name collision (`_name@N` vs `_N` vs `_@N` are disjoint), multiplicity/exhaustion, caller reachability | Not examined: none
- Basis: verified

**`to_erl_var/1` — `list_to_atom("_@"++N)` (`:70`)**
- Reason: Produces `_@N` atoms bounded by (and reused across) binding/variable count; the driving counter comes from caller-trusted eval binding size, and repeated evals reuse the same low-N atoms (no growth).
- Axes examined: counter magnitude, reuse across calls, binding size, duplicate/mixed keys (`ran lb.erl`) | Not examined: none
- Basis: verified

**`load_binding/6` + `load_pair/1` — eval binding intake (`:72-101`)**
- Reason: Only map/list bookkeeping (no atom creation beyond `to_erl_var`); operates on the application-supplied `Code.eval_*` binding (trusted caller data, not the untrusted-source attacker), and the round-trip is order/shadowing-consistent.
- Axes examined: duplicate keys (later wins), atom-key vs `{Var,Kind}` pair-key, field order, value type-opacity (`ran lb.erl`) | Not examined: none
- Basis: verified

**`dump_binding/4` — eval binding output (`:103-127`)**
- Reason: `maps:fold` with total-function guards and `maps:get(_, Binding, nil)` safe default; emits only pre-existing atom keys, creates no atoms, and reads caller-trusted post-eval binding.
- Axes examined: missing-key default, `Kind=nil` vs non-nil key selection, prune-threshold branches | Not examined: none
- Basis: verified

**`translate/4` — `lists:keyfind(version,1,Meta)` badmatch (`:20`)**
- Reason: A variable AST node lacking a `version` meta key would `badmatch`, but the expander always sets it; producing a version-less variable node requires a hand-built AST, i.e. violating the quoted-form contract (out of scope), and it is only reachable via the code-execution `quoted_to_erl`/`eval_forms` path anyway.
- Axes examined: version present/absent, `Kind=nil` vs non-nil, `'_'` fast-path | Not examined: none
- Basis: verified

**`from_env/1,2`, `assign/2,4`, `build/2` — scope construction (`:28-46,:53-70`)**
- Reason: Pure `#{}`/`map_size` bookkeeping over the internal `#elixir_erl{}`/`Macro.Env` structs supplied by in-folder callers (`elixir`, `elixir_compiler`); no external byte reaches them except via the two `list_to_atom` sinks already covered.
- Axes examined: counter seeding (`'_' => map_size`), negative-version generated vars vs non-negative source versions (collision-free), key domain | Not examined: none
- Basis: verified

### Did I verify this?

- `build_name/2` `list_to_atom` — **verified**: read `:48-49`; ran a standalone reproduction confirming `'_foo@3'` normal output and the `system_limit` crash at a 255-char name, and confirmed `list_to_atom(256)` is the underlying limit. Reachability to code-execution-only callers confirmed by grepping `elixir_erl_var:` callers (`elixir.erl:316/319/444`, `elixir_compiler.erl:51/79`). `Not examined: none`.
- `to_erl_var/1` and `load_binding`/`dump_binding` round-trip — **verified**: read `:64-127`; ran `lb.erl` exercising duplicate keys (later `99` wins → `'_@0' => 99`) and mixed atom/`{Var,Kind}` keys (kept distinct), confirming atom reuse and no round-trip corruption. Binding is caller-trusted per `elixir.erl:306-316` (`Binding` is the `eval_forms` argument). `Not examined: none`.
- `translate/4` badmatch and `from_env`/`assign`/`build` — **verified** by reading `:16-46,:53-70` and the record definitions in `elixir.hrl:54-71`; the badmatch requires a hand-built version-less AST (contract violation) and all reach here only via the executing `quoted_to_erl`/compiler callers. `Not examined: none`.

No `asserted`-basis dismissals remain; no `Depends on:` contracts are left open for the cross-file pass beyond the already-confirmed caller set.


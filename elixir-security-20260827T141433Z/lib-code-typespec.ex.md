# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/typespec.ex
<!-- sha256: bf405a7b31e749a6f81ec5a7368ea91530b6e8c39f81c5cf602a6ae601c3775a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:30:15Z_

# Security Review — `lib/elixir/lib/code/typespec.ex`

## Summary
The `binary` form of the documented `fetch_types/1`, `fetch_specs/1`, and `fetch_callbacks/1` treats a caller-supplied BEAM blob as trusted, letting a crafted `.beam` mint arbitrary atoms, dispatch to an attacker-named `backend` module, and crash the caller — a denial-of-service surface for any tool that introspects third-party BEAM files.

## Security

#### `CSCAN-typespec-untrusted-beam-debug-info`
- **Finding ID:** `CSCAN-typespec-untrusted-beam-debug-info`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `code/typespec.ex:157-175` (`typespecs_abstract_code/1`, esp. line 167 `backend.debug_info(...)` and line 160 `:beam_lib.chunks`); reached via `fetch_types/1` (`:91`), `fetch_specs/1` (`:126`), `fetch_callbacks/1` (`:147`) → `get_module_and_beam/1` (`:187`).
- **Class:** Deserialisation (primary); also Protocol/behaviour dispatch, Atom creation, Resource consumption.
- **Issue:** `fetch_types/specs/callbacks` are specced `module | binary`; when given a binary, it is parsed as a BEAM by `:beam_lib.info/1` and `:beam_lib.chunks(binary, [:debug_info])`, which decode the `Dbgi` chunk with `binary_to_term` **without `:safe`**. The decoded `{:debug_info_v1, backend, data}` is then used to call `backend.debug_info(:erlang_v1, module, data, [])` — both `backend` and `data` are fully attacker-controlled, and the `with/else -> :error` clause (`:172`) only catches pattern-match failures, not raised exceptions. Three distinct defects follow: (1) unsafe term decode mints arbitrary atoms and instantiates arbitrary terms; (2) `backend` is an arbitrary-module dispatch gadget (`M.debug_info/4` for any loaded `M`); (3) an invalid `backend` raises an uncaught `UndefinedFunctionError`.
- **Impact / threat:** Trust boundary crossed: a `.beam` from a dependency, build artifact, uploaded package, or any file the project merely reads reaches these functions (doc generators, analyzers, and plugin loaders routinely pass raw BEAM bytes rather than a loaded module atom). A crafted BEAM permanently exhausts the global atom table (atoms are never GC'd → node-global, persistent DoS), crashes the introspecting process, and forces an unexpected `debug_info/4` invocation on an attacker-named module with attacker data. Arbitrary **code execution is *not* reachable here**: the format is hard-coded to `:erlang_v1`, and both real backends (`:erl_abstract_code`, `:elixir_erl`) only run the compiler/parse-transforms on the `core_v1` path, never `erlang_v1` — which is what caps this at Medium rather than Critical. The `@moduledoc false` on the module also argues for lower weight; the node-global permanence of atom exhaustion argues for higher. A triager should weight by whether their callers feed untrusted BEAM bytes.
- **Evidence:** `verified`.
  - Backend dispatch — crafted `Dbgi` naming `EvilBackend`, then `Code.Typespec.fetch_specs(evil_beam)`:\
    `>>> EvilBackend.debug_info/4 CALLED with data={:attacker, :controlled}` (marker file written), `fetch_specs returned: :error`.
  - Uncaught DoS — `backend` set to a non-existent module: `RAISED (uncaught, DoS): function :NoSuchBackendModule646.debug_info/4 is undefined`.
  - Atom minting — hand-encoded ETF atom `etfnovel1602` (never interned in source) inside `data`: `existed BEFORE fetch_specs? false` / `exists AFTER fetch_specs? true`.
  - No-RCE bound — reading `erl_abstract_code.erl` (`debug_info(erlang_v1, …) -> {ok, AbstrCode}`) and `elixir_erl.erl` (`compile:noenv_forms` only under `core_v1`) confirms `:erlang_v1` returns forms without executing them.
- **Suggested fix:** Before dispatching, allow-list the backend (`backend in [:elixir_erl, :erl_abstract_code]`, else `:error`) and wrap the chunk read + dispatch in a `try/rescue` that returns `:error` on any raise. Document that the `binary` form must be a **trusted** BEAM — there is no `:safe` decode for `debug_info`, so untrusted bytes reaching `:beam_lib` mint atoms regardless — or restrict untrusted-input callers to the `module` (atom) form. (`erl_to_ex_var/1` at `:404`/`:407` compounds the atom pressure by minting downcased variants of BEAM-derived var names via `String.to_unsafe_atom` = `binary_to_atom`; consider `to_existing_atom` there once the input is trusted.)

### Ruled out
- **Deep-nesting recursion in `typespec_to_quoted`/`collect_vars` (DoS).** Verified linear, not amplifying: 200,000 nested `:list` nodes → 1.5 ms, no stack overflow (Erlang grows the stack on the heap). Work is O(n) in input size with no super-linear blowup, so it is not a practical DoS beyond the cost of supplying the input.
- **`spec_to_quoted/2` / `type_to_quoted/1` producing executable AST (code execution).** These build quoted type ASTs for display; they never `eval`/`compile`. Reaching code execution requires the caller to themselves `Code.eval_quoted` the result on attacker input — excluded caller-side misuse.
- **`FunctionClauseError` in `spec_to_quoted/2` on a non-`:fun`/`:bounded_fun` clause.** Reachable from crafted BEAM output, but it is the same crash/DoS class already covered by the primary finding and adds no distinct impact; `typespec_to_quoted/1` itself has a catch-all (`:371`) and does not crash on unknown nodes.
- **First-char downcasing collisions in `erl_to_ex_var/1` (identifier confusion).** Only the leading codepoint is downcased; real Erlang variables cannot start lowercase, so no two distinct Erlang vars collapse, and even a crafted collision only affects a *displayed* type AST, not any authorization/identity decision.
- **Shared mutable state / concurrency / command-exec / path / crypto / network sinks.** None present in this file (no ETS/persistent_term/process-dictionary writes, no `System`/`:os`/`File`/`Path`/`:crypto` calls); `:code.get_object_code/1` on the atom path reads a trusted, already-resolved module from the code path.

## Risks / caveats
- **`@moduledoc false`.** `Code.Typespec` is formally undocumented at the module level, though its functions carry `@doc`/`@spec` and are the de-facto introspection API used by ExDoc/IEx and third-party tooling. I treated the exported `fetch_*`/`*_to_quoted` functions as the public surface; a strict reading that the whole module is private would move the finding out of scope. Flagged rather than dropped, per the false-negative asymmetry.
- **Reachability depends on the caller.** The finding assumes some consumer passes *untrusted* BEAM bytes (dependency/artifact/upload) to the `binary` form. Callers that only pass loaded module atoms from a trusted code path are unaffected. I could not enumerate real downstream callers from this file alone.
- **beam_lib is the actual decode sink.** The unsafe `binary_to_term` executes inside OTP's `:beam_lib`, not this file; this file is what *exposes* it through a friendly `module | binary` signature with no trust caveat. There is no `:safe` option for the `debug_info` chunk, so the only real mitigations are trust-boundary documentation or backend/exception hardening as suggested.
- All behavioural claims were reproduced against `/opt/elixir` (1.21.0-dev, OTP 29), whose `code/typespec.ex` is byte-identical to the file under review (`diff` → `IDENTICAL`).


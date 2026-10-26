# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl.erl
<!-- sha256: 8aba02532d3269c6318e5d10842a61ee94035a3a18085ac31a2cf0552742805e -->

_Analysed on 2026-08-27T16:47:47Z_

# Security review: `elixir_erl.erl`

## Summary
This is the Elixir→Erlang compiler backend (translation, form generation, chunk emission); its only external-facing deserialization construct is a *generated* `binary_to_term` call on a compile-time-constant literal, and every other sink operates on trusted intra-compilation state, so there are no real security findings.

## Security

#### `CSCAN-elixir_erl-embedded-term-binary-to-term`
- **Finding ID** — `CSCAN-elixir_erl-embedded-term-binary-to-term`
- **Severity** — Low
- **Confidence** — high (that it is safe-by-construction)
- **Reachability** — `reachable` (the generated call executes in every compiled module that embeds a pid/port/reference literal — but only ever on a compile-time-constant argument; no untrusted-data path exists)
- **CVSS** — `CVSS:3.1/AV:L/AC:H/PR:H/UI:N/S:U/C:N/I:N/A:N` → 0.0 (None). *The always-flag policy nominally assigns Critical to any `binary_to_term`; I record the call site as mandated, but the true risk is None/Low because the argument is a self-produced literal — see Validation.*
- **Location** — `elixir_erl.erl:116-117`, `elixir_to_erl/2` (the `is_pid; is_port; is_reference` clause)
- **Class** — Deserialisation
- **Attack scenario** — This is reported only to satisfy the always-flag rule for `binary_to_term`. The clause fires when an Elixir AST literal being compiled is an actual pid/port/reference (e.g. a macro does `quote do: unquote(self())`). The module author already has full compile-time code execution via macros, so no attacker who is not already the author reaches this. The generated runtime call `erlang:binary_to_term(<<fixed-bytes>>)` deserializes a literal baked into the `.beam` at compile time; the byte string is `term_to_binary(Tree)` for the very `Tree` being embedded, so it can never decode to anything other than that pid/port/reference. **New capability granted by exploiting this: none** — the input is a compile-time constant, not attacker-controlled, and the module author already had code execution.
- **Trace** — `elixir_to_erl(Tree, Ann)` with `Tree` a pid/port/ref → `?remote(Ann, erlang, binary_to_term, [elixir_to_erl(term_to_binary(Tree), Ann)])`. `?remote` (`elixir.hrl:10`) expands to a `{call,…}` AST tuple, and `elixir_to_erl(term_to_binary(Tree), Ann)` (the `is_binary` clause, line 96-101) expands to a `{bin,…{string,…}…}` literal. So the emitted argument is a constant.
- **Boundary** — None crossed at runtime: the deserialized bytes originate inside the compiler, not from network/disk/caller input.
- **Impact** — Confidentiality: none. Integrity: none (round-trips to the exact embedded term). Availability: none beyond the term already residing in compiler memory.
- **Validation** — `ran erlc -o /tmp /tmp/e2e.erl && erl -noshell -pa /tmp -s e2e go`: reproduced the exact generation. Output: `Generated AST arg is a literal?  bin` and `binary_to_term(<<literal>>) => <0.10.0>  (== self()? true)` — the generated call carries a `{bin,…}` literal (bytes `131,88,119,…`) and round-trips to the embedded pid. No runtime-controllable input exists. I additionally `ran` `binary_to_term(term_to_binary(self()), [safe])` → returns the pid (OTP 29 `safe` accepts pids/refs), confirming the suggested-fix note below.
- **Suggested fix** — None required; this is the standard idiom for embedding non-literal terms into BEAM. If defense-in-depth against a corrupted literal is desired, the generated call at line 117 could target `binary_to_term/2` with `[safe]` (verified above to still accept the pid/port/ref this path reconstructs, while blocking new atoms/external funs), but it adds nothing given the argument is a compile-time constant.
- **Evidence** — `read elixir_erl.erl:116-117`; `read elixir.hrl:10` (`?remote` expands to an AST tuple, not a call); `ran` the e2e reproduction (output above).

### Ruled out

- **`term_to_binary` — checker chunk (line 148) and docs chunk (line 562, `[compressed | ChunkOpts]`)**\
  Reason — Serialization (encode) of compile-time-produced data into `"ExCk"`/`"Docs"` chunks; not a decode/deserialization sink, and any decompression/atom-exhaustion risk lives in the *reader* (`beam_lib`/`Code.fetch_docs`), not here.\
  Axes examined: direction (encode vs decode), compression flag | Not examined: reader-side decode of the produced chunk (out of file).\
  Basis — asserted.\
  Depends on: `beam_lib` / `Code` — the modules that later `binary_to_term` these chunks are responsible for safe decode of untrusted `.beam` files.

- **`erl_eval:expr/4` (line 63, `definition_to_anonymous/5`)**\
  Reason — Evaluates the module's own `def`/`defmacro` clauses into an anonymous function during compilation; running the code being compiled is by design (macros execute at compile time), so no privilege boundary is crossed.\
  Axes examined: origin of `Fun`/handlers, whether it is a sandbox claim | Not examined: none.\
  Basis — asserted.\
  Depends on: `elixir_def.erl` — supplies `Clauses`, `LocalHandler`, `ExternalHandler` from the source under compilation; a caller compiling untrusted source already accepts code execution.

- **`compile:noenv_forms` (line 34) / `elixir_erl_compiler:erl_to_core` (line 32), `debug_info(core_v1/erlang_v1, …)`**\
  Reason — Runs on `Map` extracted from a stored `{elixir_v1, Map, Specs}` debug-info term; reaching it requires reading debug_info from a crafted `.beam`, and a malformed `Map` yields only a contained function_clause crash. Loading an untrusted `.beam` is already arbitrary code.\
  Axes examined: reachability via `Code.fetch_debug_info`/`beam_lib`, crash containment (try/catch at line 34) | Not examined: full `Map`-shape fuzzing of `dynamic_form/2`.\
  Basis — asserted.\
  Depends on: `beam_lib`/`Code` — the `{elixir_v1,Map,Specs}` term is `binary_to_term`'d from a `.beam`'s `Dbgi` chunk upstream; trusting an untrusted `.beam` is out of scope.

- **`elixir_erl_compiler:noenv_forms` (line 525, `load_form/5`)**\
  Reason — Compiles the Erlang forms this module itself generated from the trusted compilation pipeline (`dynamic_form` output).\
  Axes examined: origin of `Prefix ++ Specs ++ Forms` | Not examined: none.\
  Basis — asserted.\
  Depends on: `elixir_erl_compiler.erl` — compiles forms produced by `elixir_module`/`elixir_expand`/this file.

- **`list_to_atom` via `elixir_utils:macro_name/1` (lines 264, 485)**\
  Reason — Creates one `"MACRO-"++Name` atom per `defmacro`/`defmacrop`; `Name` is already an atom from the parsed source and the count is bounded by the number of macros in the module being compiled.\
  Axes examined: unboundedness, external-vs-source origin of `Name` | Not examined: none.\
  Basis — asserted.\
  Depends on: `elixir_utils.erl:27-28` — `macro_name/1` prefixes an existing atom; atom growth is bounded by source size.

- **`elixir_utils:characters_to_list/1` (lines 196, 254, 557)**\
  Reason — Converts source-file paths (binaries) to lists; on invalid UTF-8 it raises `conversion_error` (surfaces the error, no silent replacement), satisfying UTF8_DECODE_ERRORS_SURFACED. Input is the compiler's own `RelativeFile`/`File`, not network data.\
  Axes examined: invalid UTF-8, incomplete sequence (both raise), input origin | Not examined: none.\
  Basis — verified.

- **`io_lib:format/2` in `format_error/1` (lines 686-696)**\
  Reason — Format strings are compile-time constants; arguments are callback names/arities and `Kernel.inspect` outputs. No user-controlled data reaches the format template, so no format-directive injection.\
  Axes examined: format-string constancy, `~ts` on non-list (args are inspected binaries) | Not examined: none.\
  Basis — verified.

- **ETS reads (`ets:lookup_element`/`ets:lookup`, lines 168, 591, 599, 611, 619, 628)**\
  Reason — Read the module's own data tables (`Set`) populated earlier during its compilation; intra-compilation trusted state, single-writer (`elixir_module`).\
  Axes examined: table origin, writer identity | Not examined: concurrent cross-process poisoning (tables are per-module compilation).\
  Basis — asserted.\
  Depends on: `elixir_module.erl` — owns/populates the `Set`/`Bag` tables for the module under compilation.

- **`signature_to_binary/3` → `Inspect.Algebra.format(_, infinity)` (lines 647-650)**\
  Reason — Potential `O(depth²)` layout cost, but bounded by a single function *signature* AST the module author wrote; runs only at compile time when `docs` config is true, no trust boundary crossed (self-inflicted).\
  Axes examined: width=`infinity` vs finite, signature nesting depth, compile-time-only trigger | Not examined: adversarial default-argument nesting cost curve.\
  Basis — asserted.\
  Depends on: `Elixir.Inspect.Algebra` — `format/2` cost scales with the compile-time signature the author controls (cf. memory `inspect-algebra-review`).

- **`elixir_to_erl/2` recursion on nested literals (lines 70-129)**\
  Reason — Non-tail recursion over nested tuples/lists/maps could deep-recurse, but only on the module's own literal AST at compile time; self-inflicted, no external boundary.\
  Axes examined: nesting depth, tuple/list/map/bitstring branches | Not examined: parser-imposed nesting ceiling upstream.\
  Basis — asserted.\
  Depends on: `elixir_parser`/`elixir_tokenizer` — nesting depth of literals is bounded before reaching code-gen.

- **`get_ann/1` (lines 48-56, exported)**\
  Reason — Builds `erl_anno` from an opts proplist; `line`/`column` are integer-guarded (`when is_integer(...)`), non-integers fall through the catch-all clause. No dangerous sink.\
  Axes examined: non-integer line/column, missing keys, extra keys | Not examined: negative/huge integers into `erl_anno:new` (accepted by erl_anno, no security effect).\
  Basis — verified.

### Did I verify this?

- **binary_to_term (finding)** — verified by execution; the generated argument is a `{bin,…}` literal and round-trips to the embedded pid. Kept as a Low finding per the always-flag mandate with honest severity.
- **`term_to_binary` chunks** — asserted, upgrade blocked: the risk is entirely on the reader side (`beam_lib`/`Code`), which is out of this file. Left as asserted with `Depends on: beam_lib/Code`.
- **`erl_eval:expr`** — asserted; the "by-design code execution during compilation" judgment depends on `elixir_def.erl` supplying source-derived clauses. Not upgradable from this file alone; `Depends on` retained.
- **`debug_info` core/erlang compile** — asserted; whether the `Map` can be hostile depends on `beam_lib`/`Code` decode of a `.beam`. Retained `Depends on`; not verifiable here.
- **`elixir_erl_compiler:noenv_forms` (load_form)** — the input is this file's own `dynamic_form` output (read and traced), `Not examined: none` → verified in spirit, though I list `Depends on: elixir_erl_compiler.erl` for the final compile step; the data-origin claim is verified within this file.
- **`macro_name` list_to_atom** — asserted; the atom-bound claim depends on `elixir_utils.erl:27-28` (read) plus source-size bounding. `Name` already existing as an atom is confirmed from the read; retained `Depends on` for the boundedness invariant.
- **`characters_to_list`** — read `elixir_utils.erl:138-143`; both error branches raise. No axes left → verified.
- **`io_lib:format` / `get_ann`** — read the clauses; format strings constant, line/column integer-guarded. No axes left → verified.
- **ETS reads** — asserted; single-writer/trusted-table invariant depends on `elixir_module.erl`. Retained `Depends on`.
- **`signature_to_binary` format** — asserted; the cost-boundedness depends on `Inspect.Algebra` behavior (external module + memory note). Retained `Depends on`.
- **`elixir_to_erl` recursion** — asserted; depth-bound depends on upstream parser limits. Retained `Depends on`.

## Risks / caveats
- The folder's Unicode/RFC obligations (UAX #31/#9/#15, UTS #39/#55 — bidi rejection, NFC, mixed-script, identifier/escape validation) are **not implemented in this file**; `elixir_erl.erl` is the Erlang code-gen backend and does no tokenization or identifier handling, so none of those sub-claims apply here (they live in `elixir_tokenizer.erl`/`elixir_interpolation.erl`).
- The strongest ruled-out dependencies to confirm in the cross-file pass are: (1) `beam_lib`/`Code` safely handling the `Dbgi`/`Docs`/`ExCk` chunks this file emits and that `debug_info/4` consumes, and (2) `elixir_def.erl` supplying only source-derived clauses to `definition_to_anonymous/5`. Both reflect the general principle that reaching this backend means a module is being compiled (macros run) or an untrusted `.beam` is being read — either of which is already arbitrary code by design, which is why nothing here rises above defense-in-depth.
- The single reported `binary_to_term` is safe-by-construction (compile-time literal, verified by execution) and is surfaced only to honor the always-flag policy; it is not an untrusted-deserialization vulnerability.


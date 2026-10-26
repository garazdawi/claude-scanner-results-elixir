# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/protocol.ex
<!-- sha256: 6bec90e7d0d905ced0c18f01dcf942945b53df9a0e50290555269a7bcd0807f4 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/protocol.ex
<!-- sha256: 6bec90e7d0d905ced0c18f01dcf942945b53df9a0e50290555269a7bcd0807f4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Security review — `lib/elixir/lib/protocol.ex`

## Summary
`Protocol` is compiler-support code whose documented public API (`consolidate/2`, `extract_protocols/1`, `extract_impls/2`, `assert_*`, `derive/3`) operates on already-trusted BEAM files and developer-authored AST. The one concrete sink is an unsafe `binary_to_term` on a BEAM chunk in the consolidation path; its marginal risk is small because the same call already deserializes the module's debug-info via `beam_lib`. In this re-review I re-ran the deserialization primitive in a fresh VM (executed evidence below) and read every in-file caller of the atom-creating helpers and the `derive` chain to resolve the first-pass provenance hedges.

## Security

#### `CSCAN-protocol-consolidate-unsafe-binary-to-term`
- **Finding ID** — `CSCAN-protocol-consolidate-unsafe-binary-to-term`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/elixir/lib/protocol.ex:603`, `beam_protocol/1` (reached from public `consolidate/2` at line 580, via `beam_file/1` at 622)
- **Class** — Deserialisation
- **Issue** — `beam_protocol/1` reads the `ExCk` (type-checker) chunk of the target protocol's BEAM file and passes the raw bytes to `:erlang.binary_to_term(checker)` (line 603) **without the `[:safe]` option**. Decoding of the term — including every atom it contains — happens *before* the `{@elixir_checker_version, contents}` tuple match can reject it (`with true <- is_binary(checker), {@elixir_checker_version, contents} <- :erlang.binary_to_term(checker)`), so a malformed/hostile chunk is fully materialised regardless of whether it is ultimately a valid checker payload. The `else -> nil` branch only catches the *shape* mismatch after decoding has already occurred.
- **Impact / threat** — The trust boundary is the BEAM file on disk that `consolidate/2` is pointed at. In a plain `mix compile`, those beams come from the project and its dependencies; a dependency could ship a crafted `ExCk` chunk, but that same dependency already runs arbitrary macros at compile time, so it gains nothing new here. The non-redundant threat is *build/release/analysis tooling that consolidates BEAM files from an untrusted source without executing them*: a crafted chunk encoding many distinct atoms exhausts the (never-GC'd) atom table, or a large term exhausts the heap of the tooling process — a denial of service. It is not remote code execution: `binary_to_term` does not run code, and `contents` must further be shaped like a `%{exports: ...}` map to be used downstream (`update_in(checker.exports, …)`, line 650).
- **Evidence** — `verified` (primitive + reachability), re-run this pass:
  - **Executed this re-review** — encode-in-VM-A / decode-in-fresh-VM-B, safe call isolated so no prior unsafe call pre-creates the atom:
    - `binary_to_term(Bin)` in a VM that never saw the atom → `{tag,cscan_fresh_atom_zzz9}` (atom created).
    - `binary_to_term(Bin,[safe])` alone in a fresh VM → `{'EXIT',{badarg,[{erlang,binary_to_term,[<<131,…>>,[safe]], …}]}}` (refused). This directly demonstrates the unsafe-vs-`:safe` divergence and that the missing `[:safe]` is load-bearing.
    - Method note: my first attempt showed `safe` "succeeding" only because the preceding *unsafe* call in the same VM had already interned the atom; isolating the safe call in its own VM produced the `badarg`. This is why the isolated run is the authoritative one.
  - **Read this re-review** — `beam_protocol/1` (lines 590–620) confirms the chunk is requested raw (`chunk_ids = [:debug_info, [?E,?x,?C,?k], …]`, `:allow_missing_chunks`) and that `debug_info` is returned already-decoded by `beam_lib` while the `ExCk` binary is hand-fed to `binary_to_term` — i.e. the unsafe decode is an *additional* exposure beyond `beam_lib`'s own decode, but on the same trust boundary.
  - Reachability via documented API confirmed by reading `consolidate/2` (580) → `beam_protocol/1` (590) → line 603; no guard sits between the public entry and the sink other than `is_atom(protocol)`.
  - **Still `asserted` (honestly not run):** a full malicious `.beam` (valid `debug_info` + poisoned `ExCk`) demonstrating end-to-end atom-table exhaustion was not constructed. The decode-creates-atoms primitive and the reachability are verified above; only the end-to-end amplification is unbuilt. Because `beam_lib` already decodes `debug_info` unsafely in the same function, an end-to-end PoC would not raise the severity, so I did not build it.
- **Suggested fix** — Prefer `:erlang.binary_to_term(checker, [:safe])` and treat a raised `badarg` as "no checker info" (fall through to `nil`, exactly like the existing `else -> nil`). Caveat: `ExCk` legitimately encodes cross-module type signatures whose atoms may not yet exist in the consolidating VM, so `:safe` can reject valid payloads; if that is a concern, gate decoding behind a size/format check, or accept that `consolidate/2`'s input is trusted and document it. Note this only hardens the explicit call — `beam_lib`'s own `debug_info` decode in the same function remains unsafe, so the real boundary is "only consolidate beams you trust."

### Ruled out

- **`derive/3` → `expand_callback`/`apply(mod, fun, args)`/`Module.create/3` (lines 1132–1181)** — compile-time code execution, ruled out on provenance.
  - **Axes examined:** input provenance (where `protocol`, `for`, `opts`, `mod`, `args` originate). **Not examined:** none.
  - **Verified by reading the call chain this pass:** the only in-file caller is `__derive__/3` (line 1118), `def __derive__(derives, for, %Macro.Env{} = env) when is_atom(for)`, whose `foreach` clauses further guard `proto when is_atom(proto)` and `{proto, opts} when is_atom(proto)` (lines 1120–1124). `__derive__/3` is the target of `@derive`-generated code, so `derives`/`for` are developer-authored atoms/AST from the caller's own module definition, not disk/network/decoded data. `mod`/`args` at line 1148 are `protocol` (an asserted-protocol atom, `assert_protocol!` at 1134) plus `[for, opts]` or `[for, Macro.struct!(for, env), opts]`. The code executed is the protocol author's `__deriving__` macro. No external-input path.

- **Atom creation in `__concat__/2` (`String.to_unsafe_atom`, line 1233) and the `def/1` macro (line 286)** — unbounded-atom sink, ruled out because every in-file caller feeds an already-existing atom.
  - **Axes examined:** (1) caller provenance for all in-file call sites; (2) the runtime-dispatch reachability of the `is_binary` clause. **Not examined:** none *within this file* (external callers of the `@doc false` public `Protocol.__concat__/2` are the documented trust boundary, not an in-file axis).
  - **Verified by reading this pass:** I enumerated all 15 `__concat__` references. The interesting runtime-reachable one is `struct_impl_for/1` (line 973, `Protocol.__concat__(__MODULE__, struct)`), reached from generic protocol dispatch (`impl_for(%struct{})`, line 952). `struct` there is the `__struct__` field of a runtime value; constructing such a value already requires that struct atom to exist, so it enters `__concat__` via the `is_atom` clause (1228) → `remove_prefix(Atom.to_string(struct))` → the `is_binary` clause (1232) with a binary *derived from a pre-existing atom*. `String.to_unsafe_atom` (`:erlang.binary_to_atom`, verified `string.ex:3040` in the first pass) then interns one `Elixir.Proto.Struct` atom per distinct pre-existing struct atom — bounded ~1× amplification, no unbounded growth. Every other caller (lines 363, 676, 683, 706, 748, 764, 774, 787, 935, 959, 1096, 1142, 1154, 1179) passes a literal `Any`/`"Any"`, a `built_in()` list atom, or an asserted-protocol/module atom — all compile-time-bounded. Reaching the `is_binary` clause with an *attacker-chosen fresh binary* requires calling `Protocol.__concat__/2` directly; that function is `@doc false` and is the trust boundary the module documents. `def/1` (286) is bounded by protocol-function arity at compile time.

- **`extract_protocols/1` / `extract_impls/2` file + atom handling (lines 464–542)** — directory entries come from `:file.list_dir` (real names, no caller-controlled path component past the trusted root), so no traversal; `beam_lib.chunks(file, [:attributes])` creates atoms from each beam's attributes chunk, but the input is BEAM files the caller is scanning (its own `_build`/deps ebin) and the count is bounded by file content — same "beams are trusted" boundary as the finding, weaker (nothing beyond the small attributes chunk is decoded).
  - **Axes examined:** path provenance, atom-count bound. **Not examined:** none for the in-file logic.

- **Generated `impl_for/1` / `struct_impl_for/1` dispatch (lines 946–977)** — dispatch on a value's `__struct__` is inherent to protocols; as shown in the `__concat__` entry above, the `__struct__` atom of any constructible value already exists, so runtime `__concat__` cannot create unbounded new atoms.
  - **Axes examined:** runtime atom-creation bound. **Not examined:** none.

- **`beam_file/1` fallback (lines 622–627)** — when `:code.which` returns a non-path atom the module atom is passed to `beam_lib`, which returns `{:error, …}` → `{:error, :no_beam_info}`; no dead-code or injection consequence.

- **No command execution, outbound network, crypto comparison, EEx/template, regex-from-input, `Path` traversal, ETS/persistent_term/app-env writes, or `Code.eval*`/`Module.concat` sinks** are present in the file (full read).

## Risks / caveats
- The pervasive assumption throughout this module is that **BEAM files and module atoms are trusted build inputs**. That holds for the ordinary `mix compile` consolidation flow but is worth stating explicitly for anyone repurposing `consolidate/2`/`extract_*` as a library over externally-sourced artifacts; the finding above is the concrete manifestation.
- I verified the unsafe-`binary_to_term` primitive (atom creation, `:safe` refusal) with fresh isolated runs this pass, and confirmed by reading that `consolidate/2` reaches the `ExCk` deserialization with no intervening guard. I did **not** construct a full malicious `.beam` to demonstrate atom-table exhaustion end-to-end; that single sub-step remains `asserted`, and because `beam_lib` already deserializes `debug_info` unsafely in the same function, building it would not raise the severity.
- Downstream `:elixir_erl.consolidate/4`, `Module.Types.*`, and `beam_lib` are **not in this file** and were not audited; a defect there is not attributable to `protocol.ex`. This is a scoping statement, not a dismissal of any sink in this file — the sinks this file owns are all accounted for above.

## Did I verify this?
- **Finding (unsafe `binary_to_term`)** — Primitive and reachability: **verified** (executed: isolated fresh-VM `binary_to_term`/`binary_to_term(…,[safe])` divergence; read: `beam_protocol/1`, `consolidate/2`, `beam_file/1`). End-to-end amplification PoC: **asserted** (not built, would not change severity).
- **`derive/3` ruled out** — **verified by reading** the full call chain (`__derive__/3` guards → `derive/3` → `expand_callback`); provenance is a code-reading fact, no untried axis. Upgraded from the first pass's hedged "reasoned" wording to read-verified.
- **`__concat__/2` ruled out** — **verified by reading** all 15 in-file call sites and the runtime `struct_impl_for` reachability; the bound ("one new atom per pre-existing atom") holds. Upgraded from the first pass's imprecise "requires calling directly" wording: the `is_binary` clause *is* runtime-reachable via line 973, but only with a binary derived from an already-interned atom, so the conclusion is unchanged.
- **Other ruled-out entries** — **verified by reading** the in-file logic; no external execution required.

### Dropped in re-review
- None. Every first-pass finding and ruled-out entry is preserved; the two provenance hedges (`derive/3`, `__concat__/2`) were resolved to read-verified rather than upgraded to findings, because reading the complete in-file call chains establishes that the inputs are developer atoms / pre-existing atoms with no external-input or unbounded-atom path. No assumption required upgrading to a low-confidence finding.

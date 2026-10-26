# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/protocol.ex
<!-- sha256: 1bdf277f874b23d54af11a4e95bbff361ce5f6c116935c008ea3fe341de75afd -->

_Analysed on 2026-08-27T09:28:02Z_

## Summary
`Mix.Compilers.Protocol.read_manifest/1` deserialises the on-disk protocol-consolidation manifest with `:erlang.binary_to_term/1` (no `:safe`) and then feeds the deserialised atoms into `Path.join(output, "#{atom}.beam")` file operations with no confinement, so a planted manifest yields atom-table pollution and `.beam` deletion/writes outside the consolidation directory during `mix compile`.

## Security

#### `CSCAN-protocol-manifest-unsafe-binary-to-term`
- **Finding ID** — `CSCAN-protocol-manifest-unsafe-binary-to-term`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/mix/lib/mix/compilers/protocol.ex:64`, `read_manifest/1`
- **Class** — Deserialisation (CWE-502)
- **Impact** — During `mix compile`, the consolidation manifest (`_build/<env>/lib/<app>/.mix/compile.protocols`) is read with `:erlang.binary_to_term/1` and no `:safe`. A manifest whose leading element is the current `@manifest_vsn` (3) is accepted, and every atom in its bytes is minted into the never-GC'd atom table (atom-table exhaustion, CWE-770); the `[:compressed]` payload is also transparently inflated, so a small crafted manifest can force a large allocation (decompression bomb). The deserialised atoms are the untrusted source for `CSCAN-protocol-atom-path-traversal` below. Reachability is gated on an attacker being able to plant the manifest before a build — realistically CI `_build` cache poisoning (fork PRs writing a shared cache namespace), a committed/shipped `_build`, or a co-tenant shared build dir — so this is a build-path amplifier rather than a direct dependency/network sink, hence Low. The malformed-input case is safely rescued (`try/rescue` → `clean_consolidated`), but a *well-formed* hostile term is not.
- **Evidence** — verified. Externally-built bytes containing an unknown atom are rejected by the safe decoder but minted by the form the code uses:
```
$ export PL=$(erl -noshell -eval 'B=term_to_binary([3|{0,{#{list_to_atom("cscan_only_in_erl_q")=>1},#{}}}]),io:format("~s",[binary:encode_hex(B)]),halt().')
$ elixir -e 'bin=Base.decode16!(System.fetch_env!("PL")); (try do :erlang.binary_to_term(bin,[:safe]) rescue e -> IO.puts("SAFE rejected: #{Exception.message(e)}") end); IO.puts("NON-safe: #{inspect :erlang.binary_to_term(bin)}")'
SAFE rejected: ... 1st argument: invalid or unsafe external representation of a term
NON-safe (what read_manifest uses): [3 | {0, {%{cscan_only_in_erl_q: 1}, %{}}}]
```
- **Suggested fix** — decode with `:erlang.binary_to_term(bytes, [:safe])`; the existing `try/rescue` already downgrades a rejected/unknown-atom manifest to a clean rebuild, so `:safe` is a drop-in.

#### `CSCAN-protocol-atom-path-traversal`
- **Finding ID** — `CSCAN-protocol-atom-path-traversal`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/mix/lib/mix/compilers/protocol.ex:248` (`remove_consolidated/2`) and `:171` (`consolidate_each/4`)
- **Class** — File operations and path handling (CWE-22, zip-slip-style)
- **Impact** — Both sinks build a filesystem path as `Path.join(output, "#{Atom.to_string(protocol)}.beam")` with no confinement of `protocol`. When `protocol` is an atom containing `..` (supplied via the manifest's `old_protocols`/`old_impls` maps and reached through `diff_manifest`'s removed-protocols loop, lines 230–233, in the normal incremental/`:on` compile path), the write/delete escapes the consolidation directory. Effect is limited to `.beam`-suffixed targets (the suffix is always appended), so the concrete primitive is out-of-tree deletion of `*.beam` files (e.g. clobbering other apps' compiled artifacts / forcing recompiles) and, on the write side, a `.beam`-suffixed arbitrary-file write if a genuinely consolidatable protocol can carry such a name. Adding `[:safe]` (finding above) removes the manifest-driven trigger for existing-atom traversals, but this path construction remains unconfined defence-in-depth for any dependency-derived protocol atom that reaches `consolidate_each`.
- **Evidence** — verified. Crafting the manifest exactly as `write_manifest/2` does and running the `read_manifest`→`remove_consolidated` logic verbatim deleted a file two directories above `output`:
```
victim exists before? true
deserialized old_protocols: %{"../../cscan_victim": 1}
File.rm target = /tmp/cscan_out/consolidated/../../cscan_victim.beam
victim exists AFTER?  false
```
The write side traverses identically: `Path.join(".../consolidated", "../escaped_write.beam")` created `/tmp/cscan_w/escaped_write.beam` (outside `consolidated/`).
- **Suggested fix** — confine the filename before joining, e.g. use `Path.basename("#{Atom.to_string(protocol)}.beam")`, or reject/skip protocol atoms whose string form contains a path separator or `..`, so consolidation artifacts can only ever land directly under `output`.

### Ruled out
- **Verbose-log injection via a hostile protocol atom** (`Mix.shell().info("Consolidated #{inspect_protocol(protocol)}")`, lines 174/187) — `inspect_protocol/1` routes through `Macro.inspect_atom(:literal, protocol)`, which escapes control characters (`:"evil\nFAKE-LOG-LINE\e[31m"` → `":\"evil\\nFAKE-LOG-LINE\\e[31m\""`, verified). No raw terminal/log-forging bytes reach the shell.
- **`write_manifest/2`, `clean/0`, `clean_consolidated/0`, `Code.prepend_path/1`** — all operate on fixed paths (`manifest()`, `Mix.Project.consolidation_path()`) with no external component.
- **`OptionParser.parse(args, …)` (line 16)** — `args` is the developer's own `mix` command line, not a crossed trust boundary.
- **`Task.async` + `Task.await(:infinity)` (lines 159–160)** — the task count and per-task work are bounded by real protocols/impls discovered on the code path; no attacker-controlled unbounded fan-out independent of the deserialisation DoS already noted.

## Risks / caveats
- I could not settle, from this file alone, whether a malicious **path dependency** can ship a *real, consolidatable* protocol whose module atom contains `..`/`/` such that it reaches the `File.write!` sink in `consolidate_each` (line 171) via the "new protocols" flow from `Mix.Compilers.Elixir.protocols_and_impls_from_paths/1`. That would upgrade `CSCAN-protocol-atom-path-traversal` to a dependency-driven arbitrary `.beam`-write independent of the manifest; confirming it requires reasoning about `Protocol.extract_protocols/1` and `defprotocol` naming in other files, which are out of scope here. The delete-side traversal (verified) is manifest-driven.
- Severity throughout hinges on the trust placed in `_build`. Mix uses non-`:safe` `binary_to_term` for manifests pervasively and treats `_build` as local-trusted; I rated both findings Low on that basis. If your threat model includes CI `_build` cache restore across untrusted PRs, treat the deserialisation finding as the higher-value one, since it is the entry point for the traversal.
- I exercised the sink logic with standalone scripts driving the exact file/atom/`Path.join` operations rather than a full end-to-end `mix compile` with a pre-planted manifest; the deserialisation acceptance criteria, atom minting, and both traversal directions were reproduced directly against the 1.21.0-dev/OTP-29 build under review.


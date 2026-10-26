# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/info.ex
<!-- sha256: 680f792c54eb43415982ef9bcfc6c1dda61847b1ccc39140abcd6fab16de9c20 -->

_Analysed on 2026-08-27T07:56:08Z_

I've traced every sink in the file and confirmed the behavior by running against the elixir built from this source. Here is my report.

## Summary
`IEx.Info` is an introspection/display protocol invoked explicitly by a developer through `IEx.Helpers.i/1` on a term they choose; every sink is either an injection-safe `inspect`/format, module introspection from the trusted code path, an `:erpc` call inside the distribution trust boundary, or an O(n) scan of the caller's own term — none reachable with untrusted data through a documented public API.

## Security

No findings

### Ruled out

- **`:erpc.call(node(pid), ...)` in the `PID` impl (info.ex:354) and `:erpc.call(node(port), ...)` in the `Port` impl (info.ex:395).** The node name is extracted from a pid/port the developer explicitly passes to `i/1`. A pid deserialized from attacker bytes could name an attacker node, but the auto-connect uses the local cookie and cannot complete the Erlang distribution handshake without mutual cookie knowledge; this is the established "distribution cookie = full trust" boundary, not a defect introduced here, and it only fires on an explicit interactive `i/1` on a crafted term. The call is wrapped in `try/catch`, so an unreachable/slow remote node degrades to `{"Alive", false}` / `{"Open", false}` (bounded by erpc's 5s default), not a crash. Not a finding.
- **`module?/1` → `Code.ensure_loaded?/atom` and `:code.get_object_code/1` (info.ex:89–98).** Traced the atom back to the `i/1` argument. Atoms passed as terms already exist (no atom-table growth — verified: `atom_count` delta came from the test script, not from `info/1`), and the module load/`:beam_lib.info` operate only on `.beam` files found on the operator-controlled code path. Loading a module by name is the documented purpose of `i/1` on a module atom.
- **`info_module/1`: `mod.module_info()`, `Code.fetch_docs/1`, `Protocol.extract_impls/2`, `Path.relative_to_cwd/1` (info.ex:100–162).** All operands (`:code.which`, `mod_info[:compile][:source]`, vsn/options) come from the module's own on-disk metadata on the trusted code path; the paths are only formatted for display, never used in a `File` operation.
- **`find_first_codepoint/2` via `String.codepoints/1` (info.ex:299–303) and `String.printable?/valid?` / `List.ascii_printable?` / `Keyword.keyword?` scans (info.ex:176–181, 234–237).** Content scans are O(n) with no limit and eagerly materialize the codepoint list, but measured scaling is linear (1 MB ≈ 67–120 ms, verified), and the input is the developer's own term inspected in their own shell — no cross-trust-boundary amplification. Wasteful, not a DoS.
- **String interpolations `#{inspect(first_non_printable)}` / `#{inspect(first_non_valid)}` / `#{inspect(mod)}` (info.ex:264, 280, 103, 114) and `raw_inspect` via `Inspect.Any.inspect` (info.ex:444–453, 472–481).** Every externally derived value reaches output through `inspect`, which escapes; `raw_inspect` renders a struct as a map and performs no regex compilation or code evaluation. No template/format-string sink.
- **`cond` without a `true ->` clause in the `BitString` impl (info.ex:233–238).** Exhaustive for the dispatched type: the third `is_binary/1` clause catches every binary and the fourth `is_bitstring/1` catches every non-binary bitstring, so no `CondClauseError` is reachable — verified on both a 3-bit bitstring and invalid binaries.
- **Atom creation / code execution / deserialization / path traversal / command execution.** Absent from the file — no `String.to_atom`, `Module.concat`, `binary_to_term`, `Code.eval*`, `System.cmd/shell`, `:os.cmd`, or `File`/`Path.join` writes on external components.

## Risks / caveats
- The review assumes `IEx.Info.info/1` is only ever driven by the developer's explicit `IEx.Helpers.i/1` invocation (as the moduledoc states); it is not auto-invoked by the shell on evaluation results (those use `inspect`). If a future caller were to run `info/1` automatically on network/deserialized terms, the `:erpc` auto-connect on a pid/port node name and the unbounded O(n) content scans would become worth re-rating — but that path does not exist today.
- Reproductions were run against `Elixir 1.21.0-dev` / OTP 29 built from the source under review; the latin1 locale warning is environmental and does not affect the traced behavior.


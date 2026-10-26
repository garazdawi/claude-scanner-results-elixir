# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/helpers.ex
<!-- sha256: 3f1e3b66fe0e7a7b7d4d85b6d74a17bfb1d406433f70dd6b407cd80bcaa7dc5b -->

# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/helpers.ex
<!-- sha256: 3f1e3b66fe0e7a7b7d4d85b6d74a17bfb1d406433f70dd6b407cd80bcaa7dc5b -->

_Re-reviewed on 2026-08-27_

## Summary

`IEx.Helpers` exposes interactive-shell helpers whose inputs are all supplied by the developer typing at the REPL (or the auto-loaded `.iex.exs`). No sink in this file receives data that crosses a network, archive, dependency-fetch, or build-input boundary in a way that escapes the IEx trust model. There is no in-scope security defect.

The first-pass hedges ("outside this file's boundary", "out of scope") attached to the `:erpc` remote-node calls and to the sibling-module caveats. On re-review these are not unverified assumptions about neighbour *code* — they are the standing threat model's explicit trust boundaries (distribution cluster = full trust via magic cookie; evaluator/editor/broker logic lives in other files). I re-grounded each below rather than upgrading, because the boundary is a given of the model, not an untried axis. The two executable claims from the first pass were re-run on the OTP 29 sandbox and both hold.

## Security

No findings

### Ruled out

- **`c/2` compiles arbitrary files and writes `.beam` output** (`helpers.ex:216`–`250`, `File.write!` at `237`).\
  Both the file list and destination `path` are developer-typed REPL arguments, not data from a dependency/archive. The write filename component `base` is confined to a single path segment.\
  Confidence: High.\
  Axes examined: path-traversal via the write basename (`Path.basename |> Path.rootname`), multi-extension inputs, `..`-laden inputs. Not examined: none.\
  Evidence (re-run OTP 29): `"../../etc/evil.erl" → base="evil" → /out/evil.beam`; `"/a/b/../../../etc/passwd.erl" → "passwd" → /out/passwd.beam`; `"....//x.erl" → "x"`; `"foo.beam.erl" → "foo.beam" → /out/foo.beam.beam` (rootname strips only the last extension, but the result is still a single segment — no traversal). Every case collapses to one path component under the developer-chosen `path`.

- **`pid/1,3`, `port/1,2`, `ref/1,4` parse strings via `:erlang.list_to_pid/list_to_port/list_to_ref`** (`helpers.ex:1616`–`1724`).\
  No atom-table growth and `:badarg`/`ArgumentError` on malformed input; arguments are developer-typed.\
  Confidence: High.\
  Axes examined: atom-table exhaustion under repeated parsing, malformed/garbage/empty input, out-of-range terms. Not examined: none.\
  Evidence (re-run OTP 29): atom delta = 0 over 200 interpolated `list_to_pid/port/ref` calls; `list_to_pid(~c"garbage")`, `list_to_pid(~c"<9.9.9>")`, `list_to_pid(~c"")` all return `{:error, :badarg}`. No deserialization or atom-creation sink.

- **`process_info/1` prints `$process_label` without terminal-escape sanitization** (`helpers.ex:1005`–`1007` `format_process_label/1`, printed via `print_entry`).\
  A binary `$process_label` is returned raw (`format_process_label(label) when is_binary(label) -> label`, line 1006) whereas registered names and initial calls go through `inspect` (lines 1010, 1012–1013, which escape control bytes). Terminal escapes in a raw binary label therefore survive to the terminal.\
  Ruled out: the label's source is an in-VM process the developer explicitly selects to inspect. Under the standing threat model a co-tenant BEAM process able to set a hostile `$process_label` already runs fully-trusted code with direct group-leader/terminal access, so the escape adds no escalation, and this matches the model's accepted local-trust terminal-output pattern (`Macro.Env`/source-origin content is likewise unsanitized by design). Not reached during `mix deps.get`/`compile`/`archive.install`.\
  Confidence: High (that the escape survives) / Medium (that it is non-escalating — see caveat).\
  Axes examined: raw-vs-`inspect` divergence between the binary and non-binary label clauses (confirmed by reading lines 1005–1013), reachability during dependency/build flows. Not examined: none required — the divergence is in-file and read directly, not assumed of a neighbour.

- **`r/1` recompiles the path from `module_info(:compile)[:source]`** (`helpers.ex:1041` `source_path/1`, used `496`–`523`).\
  The source path is a `.beam` attribute baked at (possibly dependency-authored) compile time, so it is externally influenced, but it is gated by `File.exists?(source)` (`502`); a dependency's compile-time absolute path essentially never exists on the victim machine (raises "could not find source"); the developer must explicitly invoke `r/ThatModule`; and the outcome is only recompiling the real on-disk source of a developer-selected module.\
  Confidence: High.\
  Axes examined: attacker-influenced source attribute, existence gate. Not examined: none.

- **`import_file` / `import_file_if_available` read and evaluate a file** (`helpers.ex:1475`–`1530`).\
  The macros require a *literal* binary path (`when is_binary(path)`, enforced at compile time; non-literal clauses raise), so no dynamic/untrusted path can be injected. This is the documented `.iex.exs` mechanism — arbitrary-code-by-design, local/operator-controlled.\
  Confidence: High.\
  Axes examined: dynamic-path injection, literal-only enforcement. Not examined: none.

- **`nl/2` and `process_info/1` issue `:erpc` calls to remote nodes** (`helpers.ex:1753`, `896`–`898`).\
  The target is `node(pid)` / a developer-supplied node, reached only through connected distributed-Erlang peers. Per the standing threat model, distribution-cluster trust is governed by the shared magic cookie (a node sharing the cookie is fully trusted and is effectively an operator); this is the model's explicitly-declared primary trust boundary, not an in-scope defense line for this file. The pid/node arguments are developer-typed at the REPL.\
  Confidence: High.\
  This entry was previously hedged as "outside this file's boundary and out of scope." Re-grounded: the boundary is a threat-model given (distribution = full trust), not an unverified claim about neighbour code, so no upgrade is warranted. Not examined: none (there is no additional axis to try — the trust is definitional).

- **`recompile/1` runs `Mix.Task.run("compile", …)`** (`helpers.ex:118`–`172`).\
  Operates on the developer's own current Mix project/install; not driven by external manifest or archive data reaching this file.\
  Confidence: High. Not examined: none.

- **`print_percentage/3` computes `trunc(min / max * 100)`** (`helpers.ex:750`–`754`).\
  `max` is a VM statistic (`:atom_limit`, `:ets_limit`, …), never zero and not attacker-influenced, so no division-by-zero is reachable.\
  Confidence: High. Not examined: none.

- **`Process.put(:iex_imported_paths, …)` and other `Process.get/put`** (`helpers.ex:1483`, `1189`, `1598`, etc.).\
  Touch only the shell process's own process dictionary — no cross-process/cross-tenant shared-state poisoning.\
  Confidence: High. Not examined: none.

## Risks / caveats

- Review is confined to this file. The genuinely dangerous IEx behaviours — arbitrary-code execution via the evaluator, `open/1` invoking `ELIXIR_EDITOR`, editor-command construction, and distribution-cluster trust — are implemented in sibling modules (`IEx.Introspection`, `IEx.Evaluator`, `IEx.Broker`, `IEx.Server`). This is explicit scope, not an assumption about their correctness: I did not audit them here and make no claim that they are safe.
- The `$process_label` raw-print is a real (if minor) terminal-escape sink. Under the stated local-trust model it is not a finding, but if the project's threat model ever treats a co-tenant BEAM process or a loaded dependency as an adversary reaching the developer terminal through debugging helpers, wrapping the binary label in `inspect/1` (as registered names already are, line 1010) would close it cheaply. I flag it as a Medium-confidence non-escalation judgement rather than a finding.
- The conclusion rests on the IEx trust model that these helpers are only ever driven by developer/operator input. If a downstream integration fed one of them (e.g. `pid/1`, `cd/1`, `process_info/1`) untrusted network data through a wrapper, that wrapper — not this file — would be the defect site.

## Did I verify this?

- **Verified by execution (OTP 29 sandbox, re-run this pass):**
  - `c/2` write-basename confinement — `Path.basename |> Path.rootname` collapses `../`, `..//`, and absolute traversal inputs to a single filename segment (4 inputs run; all confined under `/out`).
  - pid/port/ref parsing — atom delta 0 over 200 iterations; `:badarg` on `~c"garbage"`, `~c"<9.9.9>"`, `~c""`.
- **Verified by reading this file:** `format_process_label/1` returns binary labels raw while `format_registered_name/1` and `format_initial_call/1` route through `inspect` (lines 1005–1013); `c/2` File.exists? and single-segment write path (216–250); `import_file` literal-binary guard; `process_info/1` erpc targets (894–898).
- **Not verified / accepted as threat-model boundary (no upgrade — definitional, no untried axis):** distribution-peer trust for `:erpc`/`nl`; the safety of sibling modules `IEx.Introspection`/`Evaluator`/`Broker`/`Server` (explicitly out of scope, not asserted safe).

## Dropped in re-review

None. All prior ruled-out entries are preserved. No prior finding existed to drop; no hedge was upgraded to a finding, because every previously-hedged item resolves to an explicit threat-model trust boundary (distribution cluster, sibling-module scope, local-trust terminal output) rather than an untried axis or an unverified neighbour-code assumption, and the instruction to require executed evidence before upgrading is not met by any of them.

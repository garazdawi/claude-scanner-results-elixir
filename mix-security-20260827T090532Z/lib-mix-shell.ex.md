# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell.ex
<!-- sha256: 5c68206c76d8e107746cdc1e8a0f23cb8e81ecf6de126ae702a77343effc895b -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell.ex
<!-- sha256: 5c68206c76d8e107746cdc1e8a0f23cb8e81ecf6de126ae702a77343effc895b -->

_Re-analysed on 2026-08-27_

## Summary
`Mix.Shell` is a behaviour-contract module plus a thin, faithfully-documented `cmd/3` shell/argv dispatcher and a one-clause `Collectable` impl. Its only dangerous sink (`System.shell/2`, line 164) is the function's explicitly documented purpose; `command` is forwarded verbatim with no construction, interpolation, or transformation inside this file, and the forwarded option list is defensively whitelisted. No findings. All first-pass hedges concerned neighbouring modules; the two built-in shells (`shell/io.ex`, `shell/quiet.ex`) have now been read and confirm verbatim forwarding, and the remaining caveat (`deps.compile.ex`) is a caller-side decision in a different file, retained below as an explicitly-scoped pointer, not a hedged in-file finding.

## Security
No findings

### Ruled out

- **`System.shell(command, options)` command execution (line 164, `cmd/3`)** — Not a defect in this file. `command` reaches line 164 exactly as the caller supplied it: between line 145 (parameter) and line 164 there is no string operation on `command` at all — the only transforms in the function body touch `callback` (lines 146–151) and `options` (lines 153–159). The `case` at lines 162–165 only *dispatches* on shape; it never rewrites `command`. The docstring (lines 122–123) documents the string form as *"executed as a `System.shell/2` command"* and offers `{executable, args}` (line 163 → `System.cmd/3`, explicit argv, no shell) as the non-shell alternative. Reporting this would be equivalent to reporting that `System.shell/2` itself spawns a shell. Injection risk lives with whichever caller forwards untrusted bytes as a string. Confidence: high. Axes examined: dataflow into `command` (none — verbatim forward); dispatch shape; option smuggling. Not examined: none.

- **`System.cmd(command, args, options)` (line 163, `cmd/3`)** — argv is caller-supplied and forwarded unchanged; no shell interpretation. Option smuggling is blocked: line 157 `Keyword.take([:cd, :stderr_to_stdout, :env, :use_stdio])` drops any caller-supplied `:arg0`, `:into`, etc. before the list reaches `System.cmd`, and line 158 forcibly sets `:into` to a fresh `%Mix.Shell{}`. Option-injection via `-`-prefixed strings inside `args` would be a construction bug in a caller, not here. Confidence: high. Axes examined: option whitelist contents; `:into` override; argv passthrough. Not examined: none.

- **Option-list assembly (lines 153–159)** — `use_stdio` is read once (line 153) and, via `Keyword.put_new(:stderr_to_stdout, use_stdio)` (line 159), makes `:stderr_to_stdout` default to the `:use_stdio` value, matching the documented behaviour at line 134 ("`stderr_to_stdout` … defaults to `true`, unless `:use_stdio` is set to `false`"). No caller-controlled key survives except the four whitelisted ones plus the internally-forced `:into`. No weakening of any security-relevant default. Confidence: high. Axes examined: default derivation; whitelist survivorship. Not examined: none.

- **`printable_app_name/0` (lines 115–117)** — pure zero-argument delegation to `Mix.ProjectStack.printable_app_name/0`; no external input crosses into it. Confidence: high. Axes examined: argument surface (none). Not examined: none.

- **`Collectable` impl (lines 170–178)** — invokes the caller/developer-supplied `callback` per streamed `{:cont, data}` chunk and returns `:ok` for `:done`/`:halt`. The callback is developer-provided (from the built-in shells: `IO.write` in `shell/io.ex:123–126`, `fn data -> data end` in `shell/quiet.ex:64`); no attacker data crosses a trust boundary here. Confidence: high. Axes examined: callback provenance (verified against both built-in shells). Not examined: none.

- **`:quiet` callback swap (lines 146–151)** — when `options[:quiet]` is truthy the callback is replaced with `fn x -> x end`, discarding streamed output; return value is ignored by the `Collectable` clause (line 174). Matches the documented no-op intent (line 140). No security effect. Confidence: high. Axes examined: control flow; return-value use. Not examined: none.

- **`case command` clause exhaustiveness (lines 162–165)** — a `command` that is neither a 2-tuple nor a binary raises `CaseClauseError`. Reaching this requires the caller to violate the `@spec` `String.t() | {String.t(), [String.t()]}` (line 143). Crash-on-contract-violation, self-inflicted by the caller; no attacker-reachable path within this file. Confidence: high. Axes examined: caller-misuse reachability. Not examined: none.

## Risks / caveats (scoped to other files — not in-file findings)

- **Built-in shell forwarding — now verified, not assumed.** The first pass hedged that `shell/io.ex` and `shell/quiet.ex` were "read only far enough." I have now read both in full: `Mix.Shell.IO.cmd/2` (`shell/io.ex:120–127`) forwards `command` straight into `Mix.Shell.cmd/3` with an `IO.write` callback, and `Mix.Shell.Quiet.cmd/2` (`shell/quiet.ex:63–65`) forwards it with an identity callback. Neither constructs, concatenates, or interpolates `command`. This confirms — does not merely assume — that no built-in shell introduces new dataflow into the sink. (`shell/process.ex` is a test-oriented shell that captures rather than executes; not on the execution path reviewed here.)

- **`deps.compile.ex` is the file that chooses to forward dependency-controlled strings.** The security-relevant *decision* — feeding a dependency-controlled `:compile`/`make`/`erlang.mk` string into the string branch of this API — lives in `mix/tasks/deps.compile.ex`, a different file. Per the folder threat model, dependency build scripts run arbitrary code by design, so this is not a defect; but if that boundary is to be scrutinised, `deps.compile.ex` is the file to audit, not `shell.ex`. This is a pointer, not a hedged claim about `shell.ex`.

## Did I verify this?
- **Read in full:** `shell.ex` (target), `shell/io.ex`, `shell/quiet.ex` — to convert the first-pass "read only far enough" hedge into a verified statement that the built-in shells forward `command` without transformation. Verified.
- **In-file dataflow of `command`:** traced line-by-line from parameter (145) to sink (164); no transformation exists. Verified by reading.
- **Not run in the Erlang sandbox:** no execution was performed because there is no differential axis to try — `shell.ex` has a single implementation of `cmd/3` (no second implementation to push inputs through and compare), and every ruled-out entry rests on the absence of in-file dataflow rather than on a guard whose disagreement set would need probing. The `System.shell`-spawns-a-shell and `Keyword.take`-whitelist behaviours are standard, well-defined Elixir/OTP semantics. No axis remains on any entry's `Not examined` list.

## Dropped in re-review
None. The first pass reported no findings; all ruled-out entries are preserved (and firmed up with verified neighbour reads and explicit axis fields). No hedged assumption rose to the level of a low-confidence finding, because the two hedges both concerned *other* files and one of them (built-in shell forwarding) was resolved by reading those files, while the other (`deps.compile.ex`) is by construction a different file's caller decision, not a deferred claim about `shell.ex`.

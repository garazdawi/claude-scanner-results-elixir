# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/cli.ex
<!-- sha256: 97588a7eb41e99b073d433cf55ca4106b113126c856bb237a7c4afd379e6f909 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/cli.ex
<!-- sha256: 97588a7eb41e99b073d433cf55ca4106b113126c856bb237a7c4afd379e6f909 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (first pass flagged hedge phrase: "outside this file")._

## Summary

`Kernel.CLI` is the command-line / escript front-end whose inputs are operator-supplied `argv` (or, for `rpc_eval/1`, a call arriving over Erlang distribution). Every dangerous sink it reaches is driven by operator intent or gated behind the distribution full-trust boundary. In this pass I re-ran every claim the first pass had asserted against a hedge ("outside this file", "assumed"), using the local Erlang/OTP 29 + Elixir 1.21.0-dev toolchain, and confirmed each by execution. No defect crosses a trust boundary reachable through the documented public API.

The one first-pass claim that genuinely rested on the behaviour of another module (Mix's compile path) has been reframed to a claim about *this* file's contract, since neither the listed callgraph neighbours (all `src/*.erl` tokenizer files, not on cli.ex's callgraph) nor execution can settle how a third party assembles the argv this file consumes.

## Security

No findings.

### Ruled out

- **`rpc_eval/1` → `Code.eval_string(expr)` (lines 126–127) — remote code execution.** Reachable remotely only via `:erpc.call(node, Kernel.CLI, :rpc_eval, [expr])`, i.e. across Erlang distribution. `verified` by execution, not by assumption about other files:
  - Distribution is **off by default** — `elixir -e 'IO.inspect({Node.alive?(), :net_kernel.nodename()})'` printed `{false, :ignored}`, so the exported function is unreachable from any peer until an operator explicitly passes `--name`/`--sname`.
  - A peer that *can* reach `rpc_eval` already holds full RCE independent of it — `elixir -e ':erpc.call(node(), :os, :cmd, [~c"echo RCE_PROOF"])'` returned `~c"RCE_PROOF\n"`, proving `erpc` invokes any MFA. `rpc_eval` therefore grants **no incremental capability** over the standard cookie-gated distribution trust model. Not a finding about this file. Confidence: high.
- **`process_command({:eval, expr})` → `Code.eval_string(expr, [])` (lines 431–433) — `-e`/`--eval`.** `expr` is the operator's own command-line argument; evaluating it is the flag's documented purpose. `verified`: `elixir -e 'IO.puts(1+2)'` → `3`. No network/disk/dependency data reaches it. Confidence: high.
- **`List.to_unsafe_atom(node)` (line 436) — atom-table creation (CWE-770).** The atom is derived from the `--rpc-eval <node>` argument (operator-controlled), one atom per process invocation, so no unbounded growth. The value is *not* remotely influenceable: `rpc_eval/1` takes only `expr`, never a node, so no distribution peer can drive this sink. `verified`: reachable only on the `--rpc-eval` path, which errors cleanly when the node is not alive — `elixir --rpc-eval foo@bar 'IO.puts(1)'` → `--rpc-eval : Cannot run --rpc-eval if the node is not alive (set --name or --sname)`. Confidence: high.
- **`Code.require_file` / `Kernel.ParallelCompiler.require`/`compile_to_path` (lines 462, 470, 480, 491, 533) — compile-time execution.** The required/compiled targets come from `-S`, `-r`, `-pr`, the trailing file argument, or `+elixirc` patterns — every one an element of the argv this file receives. Compiling and running operator-chosen source is the tool's defined job. The first pass added a claim about Mix ("Mix compiles via `Kernel.ParallelCompiler` directly, not by re-invoking `Kernel.CLI` with dep-controlled argv"); I cannot settle another module's call pattern by reading the listed neighbours (tokenizer `src/*.erl`, not on this callgraph) or by execution, so I do not assert it. The claim scoped to *this* file stands: cli.ex treats its argv as operator intent, per its documented contract; whoever assembles that argv owns any untrusted-path exposure. Confidence: medium (scoped to this file's contract, not to Mix's behaviour).
- **`Path.expand` / `Path.wildcard` / `:filelib.ensure_path(config.output)` / `-o` (lines 388, 420–427, 503, 544–550) — path handling & traversal.** Every path component is operator command-line input; there is intentionally no `Path.safe_relative` confinement because the CLI acts on the operator's own filesystem intent (`..`, absolute paths, `**` globs expected). No external component crosses into these paths. Confidence: high.
- **`find_elixir_executable` → `:os.find_executable(file)` (line 573) — `-S` PATH search.** Standard `-S` semantics (search `$PATH`, then `Code.require_file` the match); `$PATH` is the operator's environment, and there is no shell in the path (no `os.cmd`/`System.shell`), so no command/argument injection. The `File.regular?` → `Code.require_file` TOCTOU window (lines 469–470, 580) requires an attacker with write access to a PATH/target directory, who could plant the file directly — no boundary gained. Confidence: high.
- **`Application.put_env(:elixir, :dbg_callback | :ansi_enabled, …)` (lines 35, 310, 314) — shared global writes.** Keys and values are compile-time constants selected by fixed flags (`--dbg pry`, `--color`); no externally-derived key or value is written. Confidence: high.
- **`format_error/3` → `Exception.blame` / `FunctionClauseError.blame` / `Macro.to_string` (lines 101–120) — error formatting on error terms.** The `kind`/`reason`/`stacktrace` originate from runtime-caught exceptions in the CLI/escript; the only way to control them across a boundary is the remote re-raise on line 456, which is behind the distribution full-trust boundary re-verified in the first bullet. `prune_stacktrace` (lines 214–228) recurses over a runtime-bounded stacktrace, not attacker-sized input. Confidence: high.
- **`--rpc-eval` result handling: `:erlang.raise(kind, reason, stack)` (line 456) and `Process.exit(self(), exit)` (line 453).** These act on the value returned by the remote node, trusted under the distribution model re-verified above. `verified`: the wrong-argument branch (line 296) yields `--rpc-eval : wrong number of arguments` (`elixir --rpc-eval` → that exact string) rather than a crash. Confidence: high.
- **`exec_fun/2` spawn/monitor `receive` (lines 140–178) — result spoofing / named-process race.** Matching `{^pid, res}` could in principle be forged, but only by a process already holding the freshly-spawned worker's pid inside the same VM — same-process-space, not a trust boundary. `at_exit` hooks (line 135) come from `:elixir_config`, set by the operator's own `System.at_exit`. Confidence: high.
- **Argument-injection / response-file expansion in `parse_argv`.** `verified` by execution: `parse_argv` never re-splits or `@file`-expands a single argv element; `elixir -e 'IO.inspect(System.argv())' -- 'a b c' -e DELETED` returned `["a b c", "-e", "DELETED"]`, so tokens after a registered `-e` command pass through verbatim (fallback clause, lines 392–398) and cannot be reinterpreted as options. Confidence: high.

## Risks / caveats

- **Scope of the standing folder threat model.** The Unicode/tokenizer obligations in the prompt (UAX #31/#9, UTS #39/#55, bidi/Trojan-Source, NFC, confusables) govern `elixir/lib/elixir/src/*.erl`. This file — `lib/kernel/cli.ex` — is the CLI driver and touches none of those code paths; those checks are neither implemented nor bypassable here, so they produced no findings by design, not by omission.
- **Distribution deployment posture is a deployment concern, not a file defect.** The `rpc_eval` dismissal rests on two properties I re-verified by execution this pass: distribution is off by default, and a cookie-authenticated peer already has arbitrary-MFA RCE via `erpc`. I did not (and this file cannot) audit cookie strength or `epmd` exposure — those are configured entirely outside cli.ex, and no code in cli.ex sets, weakens, or reads them. An operator who enables `--name`/`--sname` with a weak/shared cookie inherits the well-known distribution RCE exposure; cli.ex neither creates nor mitigates that, so it is not a defect attributable to this file.
- **Operator-as-attacker / host-assembled argv is out of scope.** A finding would require untrusted data (network/disk/dependency/build) to reach a sink with attacker-chosen shape. Every sink in this file is fed by operator-chosen `argv` or by a full-trust distribution peer. If a host application invokes the `elixir` executable (or an escript's `main/1`) with argv built from untrusted input, the host crosses the boundary in its own code; cli.ex is not contracted to sanitize such use and contains no code that would. This is a statement about cli.ex's documented contract, verified against its structure (every `process_command`/`parse_argv` clause consumes argv directly), not an assumption about a caller I did not read.

## Did I verify this

- **Executed** (Erlang/OTP 29, Elixir 1.21.0-dev, `ELIXIR_ERL_OPTIONS=+fnu`):
  - `{Node.alive?(), :net_kernel.nodename()}` → `{false, :ignored}` (distribution off by default).
  - `:erpc.call(node(), :os, :cmd, [~c"echo RCE_PROOF"])` → `~c"RCE_PROOF\n"` (erpc = arbitrary MFA; rpc_eval grants no incremental capability).
  - `elixir -e 'IO.puts(1+2)'` → `3` (`-e` evaluates operator arg).
  - `elixir -e '…' -- 'a b c' -e DELETED` → `["a b c", "-e", "DELETED"]` (no re-splitting / option reinterpretation of trailing tokens).
  - `elixir --rpc-eval` → `--rpc-eval : wrong number of arguments` (clean wrong-arg branch).
  - `elixir --rpc-eval foo@bar 'IO.puts(1)'` → clean "node is not alive" error (no crash on the atom/connect path).
- **Read (this file):** all of `cli.ex` (587 lines) — confirmed every `Code.eval_string`/`Code.require_file`/`ParallelCompiler`/`List.to_unsafe_atom`/`:erlang.raise` sink is fed only by argv-derived config or a distribution peer.
- **Not verifiable here (left as caveat, not asserted):** how Mix or any other host assembles the argv cli.ex consumes — not on this file's callgraph, not reachable via the listed `src/*.erl` neighbours, and not observable by execution of cli.ex alone. The corresponding first-pass Mix assertion was reframed to a this-file-contract claim (see the `Code.require_file` ruled-out entry) rather than kept as a cross-file assumption. Cookie/epmd posture likewise: configured outside this file, no cli.ex code path touches it.

## Dropped in re-review

None. Every first-pass ruled-out entry survives; the rpc_eval, `-e`, `List.to_unsafe_atom`, argv-injection, wrong-arg, and blame/distribution entries were strengthened from "verified locally / assumed" to executed evidence cited inline. No first-pass finding is dropped (there were none), and no dismissal was moved from `asserted`→`verified` without executed evidence: the only entry that retained a cross-file dependency (Mix compile path) was **narrowed in claim scope**, not upgraded in confidence, and now carries `medium` confidence bounded to this file's contract.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`process_command({:eval, expr})` → `Code.eval_string(expr, [])` (line 431–433) — `-e`/`--eval`.** `expr` is the operator's own command-line argument; evaluating it is the documented purpose of the flag (`verified`: `elixir -e 'IO.puts(1+2)'` → `3`). No network/disk/dependency data reaches it; a caller who forwards untrusted text into `-e` is the boundary they must defend, not this file.

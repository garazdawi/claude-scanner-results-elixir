# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/eval.ex
<!-- sha256: 3b4e7327d7f2e1356462b0dc39f4cd9cfc2a23a1c21b756b02c71f486a0b7767 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/eval.ex
<!-- sha256: 3b4e7327d7f2e1356462b0dc39f4cd9cfc2a23a1c21b756b02c71f486a0b7767 -->

_Re-reviewed on 2026-08-27_

`Mix.Tasks.Eval` is the `mix eval` task. Its body has exactly one dangerous sink (`Code.eval_string/1`), and the analysis turns entirely on where its input originates. I re-traced the full data path within this file (there is no cross-file hop on the path to the sink), so every "outside this file" phrase in the first pass was scoping language about hypothetical callers of a public `run/1`, not an unverified assumption about a specific callgraph neighbour.

**Trace of the one sink — `Code.eval_string(to_eval)` (line 93):**
- `to_eval` is bound from `head` (line 72), the non-option remainder of `args` produced by `OptionParser.parse_head!` (line 60).
- `args` is the argument list passed to `run/1` (line 58), the `Mix.Task` behaviour callback invoked as `Mix.Task.run("eval", args)` with the raw command-line arguments.
- The entire path from `args` to `to_eval` is contained in lines 58–72 of this file: `args` → `parse_head!` → `{_opts, head}` → `[to_eval | argv]`. No function on that path leaves the file; no network read, file read, dependency manifest, git ref, or archive entry touches it. `to_eval` is the literal string the developer types after `mix eval`.

This is the documented, intended purpose of the task (`mix eval "IO.puts(1 + 2)"`), functionally equivalent to `bash -c`. The party supplying the code string is the party who invoked the tool — no trust boundary is crossed. That is the explicitly-excluded case ("a finding that requires the victim to run `Code.eval_string` on attacker input themselves is not a finding").

I re-confirmed there are no other sinks: no `System.cmd`/`os.cmd`/shell, no atom creation, no path/`File` operations, no archive extraction, no `binary_to_term`, no network, no `Regex` compilation from input. The `System.argv(argv)` global mutation (line 91) is balanced by restoration in the `after` block (line 96), so it leaks no state even on raise.

## Summary
`Mix.Tasks.Eval` evaluates a developer-supplied command-line string via `Code.eval_string/1` — the task's documented purpose, operating on operator-controlled input with no trust boundary crossed. No security findings.

## Security

No findings

### Ruled out
- **`Code.eval_string(to_eval)` (line 93) — arbitrary code execution.**\
  Confidence: High. `to_eval` originates solely from the `mix eval` command-line argument via a fully intra-file path (`args` (line 58) → `parse_head!` (line 60) → `head` (line 71) → `to_eval` (line 72)). No dependency, config file, network peer, git ref, or archive influences it, and no function on the path escapes this file. This is the task's documented contract (evaluate the given code), equivalent to `bash -c`, and matches the explicitly-excluded "victim evals their own input" case. Constructing `mix eval "$UNTRUSTED"` in a CI script is the operator building a dangerous invocation, not a defect in this task's contract.\
  Axes examined: input provenance (CLI argv only), presence of any decode/fetch/read on the path (none), state restoration around the call. | Not examined: none.

- **`System.argv(argv)` global-state mutation (line 91).**\
  Confidence: High. VM-global write derived from CLI args, but it is intended behaviour (the eval'd code is meant to observe the passed argv) and is restored in the `after` clause (line 96) even if evaluation raises (the `try`/`after` on lines 90–97 guarantees `System.argv(old_argv)` runs on both normal return and raise). Any concurrent-reader race is self-inflicted within a single operator-driven build task, not an attacker boundary.\
  Axes examined: restoration on raise (try/after balanced), provenance of `argv`. | Not examined: none.

- **`Mix.Task.run("app.config", ["--no-app-loading" | args])` (line 75).**\
  Confidence: High. Forwards operator args to `app.config`, which evaluates the operator's own `mix.exs`/`config/*.exs`. Per the folder threat model, executing operator-controlled project config is by design (already full RCE for whoever controls those files), not a boundary this file defends. The forwarded value is the same `args` list — no untrusted source is introduced.\
  Axes examined: what data is forwarded (unmodified operator argv), whether the callee is a trust boundary (no, per threat model). | Not examined: none.

- **`"--no-mix-exs" in args` check (line 77) vs. parsed `_opts` (line 59).**\
  Confidence: High. The code tests the raw `args` list rather than the discarded `_opts` result. Minor logic redundancy, not security-relevant: both reflect the same operator-supplied flag, and the worst case is a slightly different gate on whether `mix eval` refuses to run without a `mix.exs`. No privilege or trust boundary is involved.\
  Axes examined: security relevance of the gate (none — controls only a fail-closed error path). | Not examined: none.

- **`OptionParser.parse_head!` / empty-`head` `Mix.raise` branch (lines 99–104) — dead-code check.**\
  Confidence: High. Reachable (invoking `mix eval` with no code argument), fails closed with an error message. Not dead, not a defect.\
  Axes examined: reachability, fail-closed behaviour. | Not examined: none.

## Risks / caveats
- This review is scoped to this file. `run/1` is a public `Mix.Task` callback whose contract is to receive command-line arguments; the sole intra-file path to `to_eval` carries only that CLI input. Any *other* Mix task that chose to call `Mix.Tasks.Eval.run/1` with data forwarded from an untrusted source would create a code-execution sink — but that would be a defect in that hypothetical caller, not in this file's documented API, and it is a general property of every `mix eval`-style task, not a verifiable claim about a specific existing caller. This is a scoping note, not an unverified assumption about a named neighbour.

## Did I verify this?
- I re-read the complete file (lines 1–107). Every claim about the data path to `Code.eval_string` is verified against the actual source: the path from `args` to `to_eval` is fully contained in lines 58–72 and involves no cross-file hop, so no callgraph neighbour needed reading to close it.
- I did not execute a reproduction: the only observable behaviour (evaluating a developer-supplied string) is the task's intended function, not a hostile-input decoder, traversal, or crash. There is nothing to demonstrate and no ruled-out entry with an unexamined execution axis.

### Dropped in re-review
None. All prior ruled-out entries are preserved with their reasoning intact; the first-pass "outside this file" wording was tightened to a verified intra-file trace plus an explicit scoping note, and no finding was added or removed.

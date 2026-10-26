# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io.ex
<!-- sha256: cf416b21d1eb96dd73bf29e687eb97e69842af12ae3b45e7ab8828f7b1d5236d -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io.ex
<!-- sha256: cf416b21d1eb96dd73bf29e687eb97e69842af12ae3b45e7ab8828f7b1d5236d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. The first pass reached conclusions about the downstream module `elixir_errors.erl` from timing alone ("reachable only through this public entry point", "I did not exercise the `Macro.Env` variant"). In this pass I read that neighbour directly (`elixir_errors.erl:61‑99, 214‑240`) and traced the full call path from `IO.warn/2` to `file:open`/`traverse_file_line`. Both findings are now verified at the source level, not just by timing; the two "outside this file" hedges are resolved into verified statements below._

## Summary
`IO` is almost entirely thin delegation to Erlang `:io`/`:file`; the only security‑relevant surface is `warn/2`, whose keyword clause forwards a caller‑supplied `file:`/`line:` into a snippet reader with `read_snippet: true`, yielding an undocumented arbitrary‑file read and a CPU/FD denial‑of‑service — both reachable through the documented public API. The complete downstream path (`emit_diagnostic → print_diagnostic → read_snippet → get_file_line → file:open + traverse_file_line`) was read this pass and confirmed to (a) open the caller's path and (b) loop `line` times ignoring `:eof`.

## Security

#### `CSCAN-io-warn-line-traversal-dos`
- **Finding ID:** `CSCAN-io-warn-line-traversal-dos`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `lib/elixir/lib/io.ex:380`–`402` (`warn/2`, `[{_, _} | _]` keyword clause; `read_snippet: true` at 396‑398). Defective loop is downstream at `lib/elixir/src/elixir_errors.erl:238`‑`240` (`traverse_file_line/2`), reached through `emit_diagnostic/6` (`61`‑`96`) → `read_snippet/2` (`214`‑`220`) → `get_file_line/2` (`222`‑`231`).
- **Class:** Resource consumption
- **Issue:** The keyword clause reads `line = keyword[:line]`, forms `position` from it, and passes it unbounded with `read_snippet: true`. I confirmed by reading `elixir_errors.erl` this pass: `emit_diagnostic/6` extracts `LineNumber = extract_line(Position)` (`98`: `extract_line({L,_}) -> L; extract_line(L) -> L`, so both the `{line,column}` tuple and the bare `line` reach the reader), and `get_file_line/2` calls `traverse_file_line(IoDevice, LineNumber)`. The recursive clause is `traverse_file_line(IoDevice, N) -> file:read_line(IoDevice), traverse_file_line(IoDevice, N - 1)` — it **discards the `file:read_line` result, including `:eof`**, so it iterates exactly `N` times regardless of file length. Each iteration is constant work (one `read_line` + one tail call), which is what makes cost linear in the attacker's integer. io.ex applies no upper bound (`line:` is spec'd `pos_integer()`).
- **Impact / threat:** A single `IO.warn(msg, file: any_readable_path, line: N)` spins one scheduler for `~2.9 µs × N` and holds an open FD for the duration. Trust boundary: an application/tooling path (linter, LSP, code‑review bot, diagnostic endpoint) forwarding an externally‑influenced line number into a warning's location — a spec‑valid value, not a contract violation — turns one call into a self‑amplifying DoS. The file traversal is an undocumented side effect of `line:`.
- **Evidence:** `verified` (timing + source read). Timing on a 2‑line file: `line: 200000` → `579.8 ms`; `line: 800000` → `2317.5 ms` (constant `2.9 µs/iter`, independent of file size); `[1_000_000, 4_000_000, 16_000_000]` each exceeded a 60 s timeout (exit 143). Source read this pass confirms the `:eof`‑ignoring recursion (`elixir_errors.erl:238‑240`) and the constant per‑iteration body, which is what licenses the linear extrapolation. The `line ≈ 2³¹ → ~1.5 h` figure remains an **extrapolation** from the measured constant rate (measured up to 800k iterations); I did not run the multi‑minute worst case to completion.
- **Suggested fix:** Smallest in‑file change: pass `read_snippet: false` in the keyword clause (matching `warn(message, [])` at 406 and the stacktrace clause at 411, which already use `false`), or bound/validate `line` before forwarding. Complete fix is downstream: make `traverse_file_line/2` terminate on `:eof` so cost is bounded by file length.

#### `CSCAN-io-warn-snippet-file-read`
- **Finding ID:** `CSCAN-io-warn-snippet-file-read`
- **Severity:** Low
- **Confidence:** low
- **Location:** `lib/elixir/lib/io.ex:380`–`402` (`warn/2` keyword clause; `read_snippet: true` at 396‑398). Also the `Macro.Env` clause at `372`‑`378`. File open occurs downstream at `elixir_errors.erl:222`‑`231` (`get_file_line/2`).
- **Class:** File operations and path handling
- **Issue:** `file:` is documented only as a location label. Because io.ex sets `read_snippet: true`, the renderer calls `file:open(File, [read, binary])` on that exact path and prints the requested line. I confirmed this pass that the path reaches `file:open` with no confinement: `read_snippet(File, Position)` (`218`) → `get_file_line(File, LineNumber)` (`222`) → `file:open(File, [read, binary])` (`223`). No path normalization, allow‑listing, or containment exists between the caller's string and the `open`.
- **Impact / threat:** `IO.warn(msg, file: path, line: n)` opens `path` and echoes line `n` to stderr. An application forwarding an untrusted path (e.g. `"../../etc/passwd"`) into `file:` discloses one line of an arbitrary readable file to wherever warnings are rendered. Spec type (`String.t()`) is satisfied; the read is an undocumented side effect of a label‑looking option. Limited to one line, to stderr the attacker may not observe — hence Low.
- **Evidence:** `verified` (runtime + source read). `IO.warn("something happened", file: "/tmp/secret.txt", line: 1)` printed `TOP-SECRET-LINE-1` from a plain `elixir` script (no compiler attached), confirming reachability via `print_diagnostic/2` (`emit_diagnostic/6` line `85`: `print_diagnostic(Diagnostic, ReadSnippet)` when no compiler/diagnostics buffer is active). Source read confirms the unconfined `file:open`.
- **Suggested fix:** Do not enable `read_snippet` for caller‑supplied keyword `file:`/`line:` (use `read_snippet: false`, matching the other `warn/2` clauses), or document that `file:` is opened and read at runtime so callers treat it as a trusted path.

### Ruled out
- **`Macro.Env` clause as a distinct/weaker file‑read vector (io.ex:372‑378).** *Resolved this pass — previously hedged.* I read the clause: it calls `:elixir_errors.emit_diagnostic(:warning, line, file, message, Macro.Env.stacktrace(env), read_snippet: true)`, passing the env's `line`/`file` verbatim into the **identical** downstream path as the keyword clause. So it is code‑path‑equivalent, not a separate vector — the two findings above already cover it. The only remaining difference is a data‑flow one: a `%Macro.Env{}` is normally compiler‑constructed with a trusted `file`, so reaching it with attacker‑controlled `file`/`line` requires a caller that hand‑builds or mutates the env. That makes it a weaker *reachability* story, not an unverified code path. No separate finding.
- **`binread(device, :eof)` accumulator quadratic blowup (io.ex:261‑267).** The `acc <> data` loop over 4096‑byte chunks *looks* O(n²), but measured: per‑MB cost stays flat at ~1.3–1.75 ms/MB from 32 MB→256 MB. Writable‑binary append optimization makes it amortized O(n). `verified` (timing).
- **`read(device, :eof)` / `getn_eof` unbounded read into memory (io.ex:578‑588).** Reading until `:eof` is the documented `:eof`‑mode semantics; `getn_eof` uses O(1) prepend + single reverse (linear). Unbounded memory is the caller's explicit request, equivalent to `File.read/1`. Not a defect.
- **Terminal/control‑character injection via `puts`/`write`/`warn`/`inspect`.** Writing raw chardata (incl. ANSI bytes) is the defined purpose; escaping would break legitimate formatted output. Inherent, not a defect of this file.
- **`inspect/2,3` leaking secrets via default `Inspect` (io.ex:502‑521).** Caller chooses what to inspect; io.ex adds no unsafe behavior. Out of scope (caller decision).
- **`iodata_empty?/1` returning a wrong "empty" verdict (io.ex:817‑820).** Traced all clauses (improper lists like `[1, 2 | <<3,4>>]` → integer head hits `_ -> false`); recursion is correct. No safety predicate mis‑answered.
- **`warn_once/3` writing caller keys into `:elixir_config` (io.ex:415‑424).** `@doc false`, not documented public API — excluded by scope.
- **`map_dev/1` accepting arbitrary atoms/tuples as devices (io.ex:853‑855).** A device outside `atom | pid` violates the documented `@type device`; runtime error is a caller spec violation, excluded by scope. No atom created here.
- **Atom creation / code exec / deserialization / command exec / crypto / network.** None present (no `String.to_atom`, `Module.concat`, `Code.eval*`, `binary_to_term`, `System.cmd`, `:os.cmd`, `:crypto`, or outbound requests).

## Risks / caveats
- Both findings are **contingent on a caller data flow**: the application must forward externally‑influenced values into `IO.warn/2`'s `file:`/`line:`. That is atypical (these are meant as source‑location metadata), which is why both are Low. The elevating factor is that the file‑open‑and‑read side effect is undocumented and the triggering arguments (`String.t()` path, `pos_integer()` line) are spec‑valid — so neither the "requires a spec violation" nor the "documented‑dangerous like `Code.eval`" exclusion applies. In diagnostic tooling that emits warnings at untrusted‑source‑derived locations, the DoS could reasonably be Medium.
- The **defective loop and the `file:open` live in `elixir_errors.erl`**, not io.ex. I report them here because io.ex is the documented public entry point that reaches them and the site that elects `read_snippet: true` for caller‑supplied metadata. This pass I read `elixir_errors.erl:61‑99, 214‑240` and traced the full path, so this is no longer an assumption. The minimal *in‑file* mitigation is flipping the flag; the complete fix (`traverse_file_line/2` terminating on `:eof`) is downstream.
- The `line ≈ 2³¹ → ~1.5 h` magnitude is **extrapolated** from a verified constant per‑iteration rate (2.9 µs, measured to 800k iterations) plus the confirmed constant‑work loop body; I did not run the worst case to completion.

## Did I verify this?
- **`traverse_file_line/2` ignores `:eof` → O(line) loop:** verified — read `elixir_errors.erl:233‑240`; recursive clause discards `file:read_line/1` result. Plus timing (linear, file‑size‑independent).
- **`get_file_line/2` opens the caller's path unconfined:** verified — read `elixir_errors.erl:222‑231`; `file:open(File, [read, binary])` on the raw string, no normalization. Plus runtime reproduction disclosing a file line.
- **The keyword clause forwards raw `line`/`file` with `read_snippet: true`:** verified — read `io.ex:380‑398`.
- **`extract_line/1` passes both the `{line,column}` tuple and bare `line` into the reader:** verified — read `elixir_errors.erl:98‑99`.
- **`Macro.Env` clause reaches the identical downstream path:** verified — read `io.ex:372‑378`; same `emit_diagnostic(..., read_snippet: true)` call with env `line`/`file`. (Previously hedged as "not exercised"; now resolved to a data‑flow reachability caveat, not a code‑path gap.)
- **`line ≈ 2³¹` runtime:** not verified to completion — extrapolated from measured constant rate and the confirmed constant‑work loop body.

### Dropped in re-review
None. Both prior findings are retained. The prior "Macro.Env variant not exercised" caveat was not a finding; it is resolved above (code path verified identical; reduced to a data‑flow reachability note under the existing `CSCAN-io-warn-snippet-file-read` finding).
